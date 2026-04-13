# KUtrace Kernel Porting Guide (ARM64 / Raspberry Pi 5)

This document describes **exactly** what must be added to a vanilla Linux
kernel source tree so that KUtrace can trace it. It is derived from the
working ARM64 patch at `linux-6.6.kutrace/` and was verified against a
Linux 6.12 tree in `linux/` (target: Raspberry Pi 5, kernel
`6.12.75+rpt-rpi-2712`).

KUtrace is **not** upstreamed. Every kernel release you move to, you must
re-apply these hooks by hand. The hooks themselves are tiny (a single
`kutrace1(event, arg)` call at each instrumentation point) – the work is
just finding the right spot in whatever the current source looks like.

---

## 1. How the instrumentation works

KUtrace has three pieces:

1. **Core kernel stub** – a handful of exported globals and a `kutrace_ops`
   function-pointer table. Lives in `kernel/kutrace/kutrace.c` and
   `include/linux/kutrace.h`. Built into `vmlinux` when `CONFIG_KUTRACE=y`.
2. **Hook call sites** – one-line `kutrace1(event, arg)` calls inserted at
   hot paths (syscall entry/exit, IRQ entry/exit, scheduler, page fault,
   idle, cpufreq change, TCP/UDP tx/rx, exec). Each hook is a macro that
   expands to nothing when `CONFIG_KUTRACE=n`, and to an indirect call
   through `kutrace_global_ops.kutrace_trace_1` when tracing is enabled at
   runtime.
3. **Loadable module** – `kutrace_mod.c`, built out-of-tree. On `insmod` it
   populates `kutrace_global_ops` with real trace-writing functions and
   allocates the per-CPU trace buffers. `rmmod` nulls the pointers, making
   every hook a no-op again.

The kernel never pulls the module in; the module attaches itself to the
kernel through the exported symbols. So the porting task is: **re-insert
the hook call sites and provide the core stub**. Everything else
(buffer management, postproc tools) is unchanged.

Why this scheme? The indirect call through `kutrace_global_ops` means the
hooks are "cold" when the module is absent (one load + branch), and hot
only when you actually load the module and say "go". The macro form
(`if (kutrace_tracing) ...`) short-circuits even earlier when tracing is
off at runtime but the module is loaded.

---

## 2. Files to create

### 2.1 `include/linux/kutrace.h`

Copy verbatim from `linux-6.6.kutrace/include/linux/kutrace.h`. This file
defines:

- The event number space (`KUTRACE_SYSCALL64`, `KUTRACE_IRQ`,
  `KUTRACE_TRAP`, `KUTRACE_MWAIT`, `KUTRACE_RUNNABLE`, `KUTRACE_USERPID`,
  `KUTRACE_PSTATE2`, `KUTRACE_RX_PKT`, `KUTRACE_TX_PKT`, …).
- `#define __NR_kutrace_control 1023` – the syscall number hijacked for
  user↔kernel control.
- `struct kutrace_ops` – the function-pointer table the module fills in.
- `struct kutrace_traceblock` – the per-CPU buffer descriptor.
- The `kutrace1(event, arg)`, `kutrace_pidname(p)`, `kutrace_pidrename(p)`,
  and `kutrace_pkttrace(dir, payload)` macros.
- A `#ifdef CONFIG_KUTRACE` guard so that every macro collapses to nothing
  when the feature is disabled – this is what lets the same source file
  compile with or without KUtrace.

**Why each piece matters**

- The macros, not direct calls, are what keeps the overhead at zero when
  tracing is off, and what lets you compile the same kernel with
  `CONFIG_KUTRACE=n` as a vanilla kernel.
- The `__NR_kutrace_control = 1023` trick reuses an otherwise-unused
  syscall slot so that user space can talk to the module via a plain
  `syscall(1023, cmd, arg)` without needing an ioctl or a new device node.
- The `kutrace_ops` indirection lets the module be loaded and unloaded at
  any time without rebuilding the kernel.

### 2.2 `kernel/kutrace/kutrace.c`

Copy verbatim from `linux-6.6.kutrace/kernel/kutrace/kutrace.c` (≈ 50 lines).
It defines and `EXPORT_SYMBOL`s:

```c
bool kutrace_tracing = false;
struct kutrace_ops kutrace_global_ops = {NULL, NULL, NULL, NULL};
u64 *kutrace_pid_filter = NULL;
struct kutrace_nf kutrace_net_filter = {0LLU, {0LLU, 0LLU, 0LLU}};
DEFINE_PER_CPU(struct kutrace_traceblock, kutrace_traceblock_per_cpu);
EXPORT_PER_CPU_SYMBOL(kutrace_traceblock_per_cpu);
```

These symbols are what `kutrace_mod.ko` binds to at `insmod` time. Without
the `EXPORT_SYMBOL`s the module cannot link.

### 2.3 `kernel/kutrace/Makefile`

```make
obj-$(CONFIG_KUTRACE) += kutrace.o
```

---

## 3. Files to modify

All diffs below are additions only. Line numbers drift between releases;
locate the surrounding context instead. Every hook is guarded by the
`kutrace1()` macro, which is itself `#ifdef CONFIG_KUTRACE`, so no further
ifdefs are needed at the call sites unless noted.

### 3.1 Wire `kernel/kutrace/` into the build – `kernel/Makefile`

Add next to the other feature subdirs:

```make
# KUtrace core stub (hook globals + exported symbols)
obj-$(CONFIG_KUTRACE) += kutrace/
```

### 3.2 Add the Kconfig option – `arch/arm64/Kconfig`

Near the end of the arch Kconfig, before the final `endmenu`:

```kconfig
config KUTRACE
        bool "KUtrace kernel/user tracing"
        default n
        depends on 64BIT
        depends on HZ_PERIODIC
        help
          Enables low-overhead kernel/user tracing patches.
```

`HZ_PERIODIC` is required because KUtrace samples CPU state on every
scheduler tick; `NO_HZ_IDLE` or `NO_HZ_FULL` would silently drop
samples and make the traces lie.

### 3.3 Syscall entry/exit – `arch/arm64/kernel/syscall.c`

This is the **single most important hook**. It fires on every syscall
made by every process and is the backbone of a KUtrace timeline.

Add the include at the top:

```c
#include <linux/kutrace.h>
```

Then patch `invoke_syscall()`:

```c
static void invoke_syscall(struct pt_regs *regs, unsigned int scno,
                           unsigned int sc_nr,
                           const syscall_fn_t syscall_table[])
{
        long ret;

        add_random_kstack_offset();

        if (scno < sc_nr) {
                syscall_fn_t syscall_fn;
                syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];

                /* KUtrace: entry – record syscall number + low 16 bits of arg0 */
                kutrace1(KUTRACE_SYSCALL64 | scno, regs->orig_x0 & 0xFFFFul);

                ret = __invoke_syscall(regs, syscall_fn);

                /* KUtrace: return – record syscall number + low 16 bits of retval */
                kutrace1(KUTRACE_SYSRET64 | scno, ret & 0xFFFFul);

#ifdef CONFIG_KUTRACE
        /* KUtrace: user-space control syscall (syscall nr 1023).
         * Bypasses the normal dispatch table – nr 1023 is not a real syscall. */
        } else if ((scno == __NR_kutrace_control) &&
                   (kutrace_global_ops.kutrace_trace_control != NULL)) {
                BUILD_BUG_ON_MSG(NR_syscalls > __NR_kutrace_control,
                                "__NR_kutrace_control is too small");
                BUILD_BUG_ON_MSG(16 > TASK_COMM_LEN,
                                "TASK_COMM_LEN is less than 16");
                /* arg0 = command in orig_x0, arg1 = payload in regs[1] */
                ret = (*kutrace_global_ops.kutrace_trace_control)(
                       regs->orig_x0, regs->regs[1]);
#endif
        } else {
                ret = do_ni_syscall(regs, scno);
        }

        syscall_set_return_value(current, regs, 0, ret);
        choose_random_kstack_offset(get_random_u16());
}
```

**What each part does**

- The first `kutrace1` captures *entry*: the event word is
  `KUTRACE_SYSCALL64 | scno` (base `0x800` + syscall number, so each of the
  ~450 ARM64 syscalls has a unique event id), and the payload is the low
  16 bits of the first argument. That's enough for postproc to label the
  span as "openat(fd=…)" or similar.
- The second `kutrace1` captures *return*: `KUTRACE_SYSRET64 | scno` plus
  the low 16 bits of the return value. Matching entry/return pairs is how
  the visualizer draws syscall duration bars.
- The `else if` branch is the trap door for `syscall(1023, …)`. It calls
  straight into the module's `kutrace_trace_control` handler instead of
  dispatching through the syscall table (there is no entry for 1023).
  That's how `kutrace_control` userspace tells the module to start/stop
  tracing, flush buffers, etc.
- `regs->orig_x0` holds the **original** x0 (before the syscall clobbers
  it with the return value); `regs->regs[1]` is x1. These field names are
  stable ABI in arm64 `pt_regs` – verified present in 6.12.

**6.12 note** – `__NR_compat_syscalls` was renamed to
`__NR_compat32_syscalls` in `do_el0_svc_compat()`. That is not a kutrace
concern directly, but if you copy the whole 6.6 file wholesale you will
hit a build error. Only touch `invoke_syscall()`; leave the rest alone.

### 3.4 Page fault – `arch/arm64/mm/fault.c`

```c
#include <linux/kutrace.h>
```

Inside `do_page_fault()` (or `do_mem_abort()` depending on version),
wrap the call to `__do_page_fault`:

```c
        kutrace1(KUTRACE_TRAP + KUTRACE_PAGEFAULT, 0);
        fault = __do_page_fault(mm, vma, addr, mm_flags, vm_flags, regs);
        kutrace1(KUTRACE_TRAPRET + KUTRACE_PAGEFAULT, 0);
```

Traces show page faults as their own spans – essential when diagnosing
"why is this syscall suddenly slow?" (answer: major fault).

### 3.5 Idle / WFI – `arch/arm64/kernel/idle.c`

```c
#include <linux/kutrace.h>
```

Before the `wfi()` in `arch_cpu_idle()`:

```c
        kutrace1(KUTRACE_MWAIT, 254);  /* 254 = "we went idle" marker */
        wfi();
```

This is what lets the timeline show which CPU is asleep vs busy. Without
it, an idle CPU looks identical to a CPU spinning in the kernel.

### 3.6 IPIs – `arch/arm64/kernel/smp.c`

```c
#include <linux/kutrace.h>
#define KU_IPI_BASE 246     /* IPI events mapped into IRQ space 246..252 */
```

Inside `do_handle_IPI()` (or whatever 6.12 calls it – grep for `ipi_handler`
or the `trace_ipi_entry` call):

```c
        kutrace1(KUTRACE_IRQ + ((KU_IPI_BASE + (unsigned)ipinr) & 0xFF), 0);
        /* ... existing body ... */
        kutrace1(KUTRACE_IRQRET + ((KU_IPI_BASE + (unsigned)ipinr) & 0xFF), 0);
```

IPIs are how reschedules cross CPUs; tracing them is how you see a
cross-CPU wakeup path.

### 3.7 Generic IRQ dispatch – `kernel/irq/irqdesc.c`

```c
#include <linux/kutrace.h>
```

Inside `handle_irq_desc()` (or `generic_handle_irq()` depending on
version), wrap the call:

```c
        kutrace1(KUTRACE_IRQ + (data->irq & 0xFF), 0);
        generic_handle_irq_desc(desc);
        kutrace1(KUTRACE_IRQRET + (data->irq & 0xFF), 0);
```

Optionally, just above the entry hook, sample the interrupted PC for
timer-tick profiling:

```c
        if (kutrace_tracing && (data->irq == 13))   /* 13 = arch_timer on RPi */
                kutrace_global_ops.kutrace_trace_2(
                        KUTRACE_PC, 0, instruction_pointer(get_irq_regs()));
```

This is the hook that generates the per-CPU bars in the timeline. The low
8 bits of the IRQ number pick the event color; the postproc names come
from `kutrace_control_names_rpi4.h`.

### 3.8 Soft IRQ / bottom halves – `kernel/softirq.c`

```c
#include <linux/kutrace.h>
```

In `__do_softirq()`, around the vector dispatch:

```c
        kutrace1(KUTRACE_IRQ + KUTRACE_BOTTOM_HALF, vec_nr);
        trace_softirq_entry(vec_nr);
        h->action(h);
        trace_softirq_exit(vec_nr);
        kutrace1(KUTRACE_IRQRET + KUTRACE_BOTTOM_HALF, 0);
```

Softirqs look like IRQs in the trace but carry the softirq vector number
(net-rx, timer, tasklet…) as payload. Without this you lose visibility
into network bottom halves.

### 3.9 Scheduler – `kernel/sched/core.c`

```c
#include <linux/kutrace.h>
```

Four hook sites, all important:

**A. `try_to_wake_up()` – target became runnable (two call sites, one for the
remote-wakeup path, one for the local path). Add before `ttwu_do_wakeup()` /
`ttwu_do_activate()`:**

```c
        kutrace1(KUTRACE_RUNNABLE, p->pid);
```

**B. `__schedule()` – entry, treat the scheduler itself as a fake
"syscall 511" so it appears as a bracketed span in the trace:**

```c
        kutrace1(KUTRACE_SYSCALL64 + KUTRACE_SCHEDSYSCALL, 0);
        cpu = smp_processor_id();
        /* ... existing __schedule body ... */
```

**C. `__schedule()` – just before `context_switch()`, record the new
pid so the timeline can color the CPU bar by task:**

```c
        kutrace_pidname(next);                 /* emit comm name first time seen */
        kutrace1(KUTRACE_USERPID, next->pid);
        /* fall through to context_switch(rq, prev, next, &rf); */
```

**D. `__schedule()` – the early return path when `prev == next` (no
switch). Close the fake syscall span:**

```c
        kutrace1(KUTRACE_SYSRET64 + KUTRACE_SCHEDSYSCALL, 0);
```

The trick here is that wrapping `__schedule()` in a fake syscall pair
means the visualizer draws scheduler time as a distinct span, so you can
see context-switch cost directly. The `KUTRACE_RUNNABLE` event plus the
`KUTRACE_USERPID` event are what produce the "wakeup arrow" connecting
a waker CPU to a wakee CPU in the timeline.

### 3.10 Exec / PID rename – `fs/exec.c`

```c
#include <linux/kutrace.h>
```

In `begin_new_exec()` (after the comm has been set), unconditionally
re-publish the pid→name mapping because execve changes it:

```c
        kutrace_pidrename(current);
```

Without this, every `execve()` leaves the trace showing the *old* comm
for the task until the next fork.

### 3.11 CPU-frequency changes – `drivers/cpufreq/cpufreq.c` and `drivers/cpufreq/cppc_cpufreq.c`

```c
#include <linux/kutrace.h>
```

In `cpufreq_notify_transition()` (and in `cppc_cpufreq_set_target()` on
RPi5 which uses cppc), right where the new frequency becomes effective:

```c
        kutrace1(KUTRACE_PSTATE2, freqs->new / 1000);   /* MHz */
```

P-state tracing lets the visualizer explain "why did this loop suddenly
take 2× longer?" with a frequency-change event.

### 3.12 TCP/UDP payload tracing – `net/ipv4/tcp_input.c`, `tcp_output.c`, `udp.c`

```c
#include <linux/kutrace.h>
```

In each RX/TX path, just before handing the packet off:

```c
#ifdef CONFIG_KUTRACE
        if (kutrace_tracing && ((hdrlen + 32) <= len)) {
                const u64 *ku_payload = (u64 *)(skb->data + hdrlen);
                kutrace_pkttrace(KUTRACE_RX_PKT, ku_payload);   /* or TX */
        }
#endif
```

`kutrace_pkttrace` hashes the first 32 bytes of payload and only inserts
the event if the hash passes the filter configured from user space.
That's how KUtrace correlates packets across machines without logging
every packet. These blocks are wrapped in `#ifdef CONFIG_KUTRACE`
explicitly (not just the macro) because they declare local variables and
dereference the skb.

---

## 4. Building the module out-of-tree

Once the kernel is patched and built with `CONFIG_KUTRACE=y`, build
`kutrace_mod.c` against it:

```
cd KUtrace/linux/module
make -C /lib/modules/$(uname -r)/build M=$PWD modules
sudo insmod kutrace_mod.ko tracemb=20
```

The module calls `EXPORT_SYMBOL`'d globals from `kernel/kutrace/kutrace.c`
(`kutrace_tracing`, `kutrace_global_ops`, `kutrace_pid_filter`,
`kutrace_net_filter`, `kutrace_traceblock_per_cpu`). If any of those
symbols is missing from `/proc/kallsyms` after boot, the kernel patch is
incomplete.

---

## 5. Porting checklist when moving to a new kernel version

1. Copy `include/linux/kutrace.h`, `kernel/kutrace/kutrace.c`,
   `kernel/kutrace/Makefile` unchanged.
2. `obj-$(CONFIG_KUTRACE) += kutrace/` in `kernel/Makefile`.
3. `config KUTRACE` entry in `arch/arm64/Kconfig`.
4. For each file listed in §3, grep for the surrounding function name
   (not line number – they drift) and re-insert the hook. The ARM64
   `pt_regs` field names (`orig_x0`, `regs[1]`, `regs[7]`, `regs[8]`) are
   stable ABI and have not changed 6.6 → 6.12.
5. Watch for renames: 6.12 renamed `__NR_compat_syscalls` →
   `__NR_compat32_syscalls`. Similar single-identifier churn is the main
   failure mode between releases.
6. `make` the kernel. The hooks will either compile cleanly or fail
   loudly – there is no subtle middle ground because every hook is a
   single macro call.
7. Build and `insmod` the module; run `kutrace_control go … stop` and
   confirm a non-empty trace.

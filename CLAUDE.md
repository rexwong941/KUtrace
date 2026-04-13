#  CLAUDE.md

Guidance for agents working in this repository.

## What this repo is

KUtrace — a very-low-overhead kernel/user tracing facility from Richard Sites' *Understanding Software Dynamics*. This fork is the **Raspberry Pi 5 / Linux 6.12.80** port. Upstream supports x86, RPi4, RISC-V, Android, FreeBSD; the active target here is **RPi5 (arm64, kernel `6.12.80-v8-16k-kutrace+`)**.

KUtrace has three pieces:
1. **Core kernel stub** — globals + `kutrace_ops` fn-ptr table, built into `vmlinux` when `CONFIG_KUTRACE=y`.
2. **Hook call sites** — one-line `kutrace1(event, arg)` macros at syscall/IRQ/sched/pagefault/idle/cpufreq/TCP/UDP/exec paths. Zero-overhead when tracing off.
3. **Loadable module** `kutrace_mod.ko` — populates `kutrace_global_ops` on `insmod`, owns per-CPU trace buffers. `rmmod` nulls the pointers → hooks become no-ops.

## Skills

- `.claude/skills/kutrace-tracing/` — invoke when the task is to capture a trace and produce an HTML timeline. Handles module load, `kutrace_control` capture variants, and `postproc3.sh`.

## Repo layout

- `RPI5_BUILD.md` — **read first.** End-to-end RPi5 build: host prereqs, kernel patch, `CONFIG_KUTRACE`, kernel build/install, out-of-tree module, postproc, smoke test, rebuild cheatsheet.
- `RPI5_TRACING.md` — quickstart for loading the module and capturing a trace.
- `linux/RPI5_KERNEL_PORTING.md` — authoritative per-hook-site porting guide (what to add to each kernel file and *why*). Use this when moving to a new kernel version.
- `linux/patches-linux-6.12.80-rpi5.tar.gz` + `linux/patches-6.12.80-rpi5/` — working `.original`/`.patched` pairs for the RPi5 6.12.80 port. Diff these when locating hook sites in a newer kernel.
- `linux/module/` — out-of-tree `kutrace_mod.c` + Makefile. Build with `make -C /lib/modules/$(uname -r)/build M=$PWD modules`.
- `linux/control/` — `kutrace_control` userspace source.
- `linux/linux-6.6.36/`, `linux/linux-6.1.12/` — reference patched kernel trees for older versions.
- `linux/android/` — Android (Pixel 6 Pro / GKI) port notes; not active on RPi5.
- `postproc/` — `rawtoevent`, `eventtospan3`, `spantotrim`, `samptoname_{k,u}`, `kutrace_control`, `postproc3.sh`, `build_postproc.sh`. Produces `.json` + `.html` timeline.
- `docs/` — `kutrace_user_guide.pdf`, `linux_patches_installation_guide.pdf` (authoritative user-facing docs).
- `bookcode/`, `bookfigs/` — example programs and HTML figures from the *Understanding Software Dynamics* book.
- `freebsd/`, `riscv_linux/` — other-OS ports; not touched for RPi5 work.

## Assumed paths on this machine

- Kernel tree: `/home/pi/linux`
- KUtrace userspace: `/home/pi/KUtrace` (and this checkout at `/home/pi/kutrace/KUtrace.rpi5`)
- Running kernel: `6.12.80-v8-16k-kutrace+`

## Common workflows

**Rebuild module only** (after editing `linux/module/kutrace_mod.c`):
```
sudo rmmod kutrace_mod
cd linux/module && make -C /lib/modules/$(uname -r)/build M=$PWD modules
sudo insmod kutrace_mod.ko tracemb=64
```

**Rebuild kernel** (after editing hook sites):
```
cd /home/pi/linux
KERNEL=kernel_2712 make -j4 Image.gz modules dtbs
sudo make -j4 modules_install
sudo cp arch/arm64/boot/Image.gz /boot/firmware/kernel_2712.img
sudo reboot
```

**Capture a trace**:
```
cd postproc
printf "init\ngo\nwait 2\nstop smoke\nquit\n" | sudo ./kutrace_control
./postproc3.sh smoke "kutrace smoke test"   # -> smoke.{trace,json,html}
```

## Rules for agents

- **`include/linux/kutrace.h` is shared ABI between kernel and module.** Changing `struct kutrace_traceblock` (or any layout) requires rebuilding *both* sides. Mismatch causes silent memory corruption.
- `struct kutrace_traceblock` must contain `prior_llc_misses` — the module references it even though LLC tracking is a no-op on RPi.
- On RPi, `ku_setup_llc_miss()` / `ku_get_llc_miss()` use an `IsRPi4_64` branch (LLC counters unimplemented on ARM — don't re-add x86 PMU code).
- Kernel config must be `CONFIG_KUTRACE=y` **and** `CONFIG_HZ_PERIODIC=y`. NO_HZ drops scheduler-tick samples and silently corrupts traces.
- Syscall 1023 (`__NR_kutrace_control`) is the user↔module channel. Do not reassign it.
- When porting hooks to a new kernel, **locate by surrounding function name, not line number** — line numbers drift. `pt_regs` arm64 field names (`orig_x0`, `regs[1]`, …) are stable ABI.
- The upstream book-origin files (`bookcode/`, `bookfigs/`, `docs/`, `freebsd/`, `riscv_linux/`) are reference material — don't modify unless the task is explicitly about them.
- Don't commit build artifacts (`*.o`, `*.ko`, `*.mod*`, `.*.cmd`, `Module.symvers`, `modules.order`, or the stripped binaries in `postproc/`). Several are already untracked in the working tree — leave them that way unless asked.
- Destructive kernel ops (flashing `/boot/firmware/kernel_2712.img`, `rmmod`ing a busy module, reboots) must be confirmed with the user before running.

## Where to look first for a given task

| Task | Start here |
|---|---|
| "Build / install on RPi5" | `RPI5_BUILD.md` |
| "Capture a trace / use it" | skill `kutrace-tracing`, `RPI5_TRACING.md`, `docs/kutrace_user_guide.pdf` |
| "Port to a different kernel" | `linux/RPI5_KERNEL_PORTING.md` + diff `linux/patches-6.12.80-rpi5/` |
| "Module won't load / symbol missing" | `linux/RPI5_KERNEL_PORTING.md` §2.2 (EXPORT_SYMBOL list) |
| "Postproc / visualization" | `postproc/postproc3.sh`, `postproc/build_postproc.sh` |
| "Event names / IRQ names" | `postproc/kutrace_control_names_rpi4.h` |

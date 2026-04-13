# KUtrace Build Procedure — Linux 6.12 / Raspberry Pi 5

Complete build procedure for the kutrace-enabled kernel, the out-of-tree
kutrace module, and the userspace postproc tools.

Paths assumed:
- Kernel tree:        `/home/pi/linux`
- Kutrace userspace:  `/home/pi/KUtrace`

---

## 1. Host prerequisites (one time)

```bash
sudo apt install -y bc bison flex libssl-dev make libc6-dev libncurses5-dev \
                    build-essential git
```

---

## 2. Kernel source patches

Decide first whether the target kernel matches the known-good RPi5 patch set.

Check the running RPi5 kernel version:

```bash
uname -r
```

If the target kernel is Linux `6.12.80` for Raspberry Pi 5, use the working
reference modification already in this repo:

- `linux/patches-linux-6.12.80-rpi5.tar.gz`

If the target kernel does not match `6.12.80`, an agent should still be able
to port it by following this process:

1. Read `linux/RPI5_KERNEL_PORTING.md`.
2. Extract `linux/patches-linux-6.12.80-rpi5.tar.gz` and use its
   `.original`/`.patched` pairs as the concrete example of a working RPi5
   port.
3. Recreate the same hook sites and support files in the new kernel tree,
   adjusting for source drift in the newer kernel.

The target kernel tree should end up containing:

**New files**
- `include/linux/kutrace.h`   — event constants, `kutrace_ops`, `kutrace1()` macros
- `kernel/kutrace/kutrace.c`  — EXPORT_SYMBOL stubs for globals + fn-ptr table
- `kernel/kutrace/Makefile`   — `obj-$(CONFIG_KUTRACE) += kutrace.o`

**Edits**
- `kernel/Makefile`                      — `obj-$(CONFIG_KUTRACE) += kutrace/`
- `arch/arm64/Kconfig`                   — `config KUTRACE` bool option
- `arch/arm64/kernel/syscall.c`          — SYSCALL64/SYSRET64 hooks + `__NR_kutrace_control` dispatch
- `arch/arm64/kernel/idle.c`             — MWAIT hook in `cpu_do_idle`
- `arch/arm64/kernel/smp.c`              — IPI IRQ/IRQRET hooks in `do_handle_IPI`
- `arch/arm64/mm/fault.c`                — TRAP/TRAPRET hooks around `handle_mm_fault`
- `kernel/softirq.c`                     — BOTTOM_HALF IRQ/IRQRET hooks
- `kernel/irq/irqdesc.c`                 — IRQ/IRQRET hooks + PC sampling
- `kernel/sched/core.c`                  — RUNNABLE, SCHEDSYSCALL, USERPID hooks
- `fs/exec.c`                            — `kutrace_pidrename()` after execve
- `drivers/cpufreq/cpufreq.c`            — `KUTRACE_PSTATE2` in PRECHANGE
- `drivers/cpufreq/cppc_cpufreq.c`       — `KUTRACE_PSTATE2` before transition_begin
- `net/ipv4/tcp_input.c`                 — RX_PKT filter in `tcp_rcv_established`
- `net/ipv4/tcp_output.c`                — TX_PKT filter in `__tcp_transmit_skb`
- `net/ipv4/udp.c`                       — TX_PKT in `udp_send_skb`, RX_PKT in `__udp4_lib_rcv`

`linux/RPI5_KERNEL_PORTING.md` gives the exact hook intent and insertion
pattern for each file. `linux/patches-linux-6.12.80-rpi5.tar.gz` provides the
working `.original`/`.patched` examples to diff against when locating the
corresponding sites in a newer kernel.

**Note**: `struct kutrace_traceblock` in `include/linux/kutrace.h` must contain
`prior_llc_misses` (the kutrace module references it).

---

## 3. Kernel configuration

```bash
cd /home/pi/linux
KERNEL=kernel_2712 make bcm2712_defconfig
./scripts/config --enable KUTRACE
./scripts/config --set-str LOCALVERSION "-v8-16k-kutrace"
```

Verify:

```bash
grep -E 'KUTRACE|LOCALVERSION' .config
# CONFIG_LOCALVERSION="-v8-16k-kutrace"
# CONFIG_KUTRACE=y
```

---

## 4. Kernel build

```bash
cd /home/pi/linux
KERNEL=kernel_2712 make -j4 Image.gz modules dtbs
```

Artifacts produced:
- `arch/arm64/boot/Image.gz`
- `arch/arm64/boot/dts/broadcom/*.dtb`
- `arch/arm64/boot/dts/overlays/*.dtbo`
- in-tree `.ko` modules under the tree

---

## 5. Kernel install

```bash
cd /home/pi/linux
sudo make -j4 modules_install

sudo cp /boot/firmware/kernel_2712.img /boot/firmware/kernel_2712.img.bak
sudo cp arch/arm64/boot/Image.gz                  /boot/firmware/kernel_2712.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb        /boot/firmware/
sudo cp arch/arm64/boot/dts/overlays/*.dtb*       /boot/firmware/overlays/
sudo cp arch/arm64/boot/dts/overlays/README       /boot/firmware/overlays/
```

Reboot and verify:
```bash
sudo reboot
# after reboot:
uname -a          # expect: 6.12.80-v8-16k-kutrace+
ls /lib/modules/$(uname -r)/build   # headers available for out-of-tree build
```

---

## 6. Out-of-tree kutrace module

Source: `/home/pi/KUtrace/linux/module/kutrace_mod.c`

### 6a. RPi-specific LLC stubs

The upstream module has `#error` guards for LLC-miss counters that are not
implemented on ARM. Add `IsRPi4_64` branches in `ku_setup_llc_miss()` and
`ku_get_llc_miss()` so they compile cleanly (LLC tracking is a no-op on RPi):

```c
void ku_setup_llc_miss(void) {
#if IsIntel_64
    ...
#elif IsRPi4_64
    /* LLC tracking not implemented on RPi */
#else
#error Define ku_setup_llc_miss for your architecture
#endif
}

inline u64 ku_get_llc_miss(void) {
#if IsIntel_64
    ...
#elif IsRPi4_64
    return 0;
#else
#error Define llc_miss for your architecture
    return 0;
#endif
}
```

### 6b. Build

```bash
cd /home/pi/KUtrace/linux/module
make -C /lib/modules/$(uname -r)/build M=$PWD modules
```

Produces `kutrace_mod.ko` (~27 KB).

### 6c. Load

```bash
sudo insmod kutrace_mod.ko tracemb=64    # 64 MB buffer (default 2)
lsmod | grep kutrace
dmesg | tail
# expect:
#   kutrace_trace hello =====================
#   vmalloc kutrace_pid_filter ...
#   vmalloc kutrace_tracebase(64 MB) ... OK
#   IsRPi4_64
#   kutrace_trace All done init successfully!
```

**Module parameters** (`module_param`):

| name      | default | meaning                                  |
|-----------|---------|------------------------------------------|
| `tracemb` | 2       | MB of kernel trace buffer to vmalloc     |
| `check`   | 1       | require PTRACE permission (0 = disable)  |
| `pktmask` | —       | packet-trace filter mask                 |
| `pktmatch`| —       | packet-trace filter match                |

Inspect at runtime: `cat /sys/module/kutrace_mod/parameters/tracemb`.

To change the buffer size after loading, unload and reload:

```bash
sudo rmmod kutrace_mod
sudo insmod kutrace_mod.ko tracemb=128
```

### 6d. Unload

```bash
sudo rmmod kutrace_mod
```

### 6e. Optional: persistent install

```bash
sudo mkdir -p /lib/modules/$(uname -r)/extra
sudo cp kutrace_mod.ko /lib/modules/$(uname -r)/extra/
sudo depmod -a
sudo modprobe kutrace_mod tracemb=64
```

---

## 7. Postproc tools (userspace)

The prebuilt aarch64 binary `kutrace_control` already exists at
`/home/pi/KUtrace/postproc/kutrace_control`. To rebuild from source:

```bash
cd /home/pi/KUtrace/postproc
./build_postproc.sh
```

This produces: `kutrace_control`, `rawtoevent`, `eventtospan3`,
`spantotrim`, `samptoname_k`, `samptoname_u`, `makeself`, etc.

---

## 8. Quick end-to-end smoke test

```bash
cd /home/pi/KUtrace/postproc
printf "init\ngo\nwait 2\nstop smoke\nquit\n" | sudo ./kutrace_control
./postproc3.sh smoke "kutrace smoke test"
# produces smoke.trace, smoke.json, smoke.html
```

Open `smoke.html` in a browser to view the timeline.

---

## 9. Rebuild cheatsheet

When editing kernel hook sites:

```bash
cd /home/pi/linux
KERNEL=kernel_2712 make -j4 Image.gz modules dtbs
sudo make -j4 modules_install
sudo cp arch/arm64/boot/Image.gz /boot/firmware/kernel_2712.img
sudo reboot
```

When editing only the out-of-tree module:

```bash
sudo rmmod kutrace_mod
cd /home/pi/KUtrace/linux/module
make -C /lib/modules/$(uname -r)/build M=$PWD modules
sudo insmod kutrace_mod.ko
```

When editing `include/linux/kutrace.h` (shared struct/ABI):
- **Must rebuild both kernel AND module** — the struct layout is compiled
  into both sides. Mismatch causes silent memory corruption.

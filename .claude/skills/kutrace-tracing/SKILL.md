---
name: kutrace-tracing
description: Capture a KUtrace trace on Raspberry Pi 5 and post-process it into an HTML timeline. Use when the user asks to "trace", "capture a kutrace", "profile with kutrace", produce a `.trace`/`.json`/`.html` timeline, or investigate latency/scheduling/IRQ behavior on this RPi5 kernel.
---

# KUtrace tracing on Raspberry Pi 5

This skill runs an end-to-end KUtrace capture on the RPi5 build in this repo and post-processes the result into an HTML timeline. It is the operational companion to `RPI5_TRACING.md`.

## Prerequisites (verify before tracing)

Run these checks first. If any fail, stop and report to the user — do **not** try to rebuild the kernel or module without permission.

```bash
uname -r                      # expect: 6.12.80-v8-16k-kutrace+ (or similar -kutrace)
ls linux/module/kutrace_mod.ko
ls postproc/kutrace_control postproc/postproc3.sh
```

If `kutrace_mod.ko` is missing, build it:
```bash
cd linux/module
make -C /lib/modules/$(uname -r)/build M=$PWD modules
```

If `postproc/kutrace_control` is missing, build postproc:
```bash
cd postproc && ./build_postproc.sh
```

## 1. Load the module

```bash
lsmod | grep -q kutrace_mod || sudo insmod linux/module/kutrace_mod.ko tracemb=64
dmesg | tail -n 20            # expect: "All done init successfully!"
```

Module parameters:
| param     | default | meaning                               |
|-----------|---------|---------------------------------------|
| `tracemb` | 2       | MB of trace buffer (use 64 for real runs) |
| `check`   | 1       | require PTRACE (0 disables)           |
| `pktmask` / `pktmatch` | — | packet-trace filter             |

To change buffer size: `sudo rmmod kutrace_mod && sudo insmod kutrace_mod.ko tracemb=N`.

## 2. Capture a trace

Pick one form. The capture lands `<name>.trace` in the current directory.

**Scripted (non-interactive)** — preferred for agent use:
```bash
cd postproc
printf "init\ngo\nwait 2\nstop <name>\nquit\n" | sudo ./kutrace_control
```
Replace `wait 2` with the number of seconds you want to trace, and `<name>` with the run label (e.g. `smoke`, `build_latency_01`).

**Interactive** (when the user wants to drive it):
```
sudo ./kutrace_control
  control> init
  control> go          # or: goipc / gowrap / goipcwrap
  control> wait 5
  control> stop myrun
  control> quit
```

Command variants (pick based on user intent):
- `go` — plain trace.
- `goipc` — include instructions-per-cycle per event (recommended for most analysis).
- `gowrap` — wrap-around buffer (keep only the *last* N MB; use when the event of interest is at the end of a long run).
- `goipcwrap` — both.

**One-shot on/off** (when you want to wrap trace around an external command):
```bash
sudo ./kutrace_control 1     # start
<run workload>
sudo ./kutrace_control 0     # stop  (writes ku_<timestamp>.trace)
```

## 3. Post-process to HTML timeline

```bash
cd postproc
./postproc3.sh <name> "<title string shown in timeline>"
```

Produces `<name>.json` and `<name>.html` next to `<name>.trace`. Open `<name>.html` in Chromium to view. On a headless session, tell the user the file path — don't try to open a browser.

`postproc3.sh` chains `rawtoevent` → `eventtospan3` → `samptoname_{k,u}` → timeline HTML. If it fails, run the stages individually from `postproc/` to locate the break.

## 4. Unload the module (optional)

Only if the user asked to clean up, or if you need to change `tracemb`:
```bash
sudo rmmod kutrace_mod
```
Do **not** unload by default — leaving it loaded costs nothing when tracing is off.

## Rules

- **Always `sudo`** for `insmod`, `rmmod`, and `kutrace_control`. The control syscall (1023) requires it.
- **Never rebuild the kernel** from this skill. If hooks look missing (e.g. `dmesg` shows unresolved symbols, or traces come back empty), stop and report — kernel rebuilds need explicit user approval and a reboot.
- **Confirm before `rmmod`** if any other process might be mid-trace. `lsmod` shows refcount; nonzero means someone is using it.
- **Trace files are large.** A 64 MB buffer produces a multi-MB `.trace`. Don't commit them. Put runs somewhere under `/tmp` or a scratch dir unless the user specifies a location.
- **Don't invent run names.** Ask the user or use `smoke` for a quick verification.
- If `dmesg` shows `IsRPi4_64` during init, that's expected — the RPi5 build reuses the RPi4 LLC stubs.

## Quick smoke test (end-to-end verification)

Use this exact sequence to confirm everything works after a module rebuild or reboot:
```bash
sudo insmod linux/module/kutrace_mod.ko tracemb=64 2>/dev/null || true
cd postproc
printf "init\ngo\nwait 2\nstop smoke\nquit\n" | sudo ./kutrace_control
./postproc3.sh smoke "kutrace smoke test"
ls -la smoke.trace smoke.json smoke.html
```
A successful run produces all three files and `smoke.html` is non-empty.

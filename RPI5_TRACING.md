# KUtrace on Raspberry Pi 5

## 1. Load module
```bash
cd /home/pi/KUtrace/linux/module
sudo insmod kutrace_mod.ko tracemb=64    # 64 MB trace buffer (default 2)
lsmod | grep kutrace
dmesg | tail        # expect: "All done init successfully!"
```

Module parameters: `tracemb=N` (MB of buffer, default 2), `check=0` (disable
PTRACE check), `pktmask`/`pktmatch` (packet-trace filter).
Inspect: `cat /sys/module/kutrace_mod/parameters/tracemb`

## 2. Capture a trace
```bash
cd /home/pi/KUtrace/postproc
printf "init\ngo\nwait 2\nstop smoke\nquit\n" | sudo ./kutrace_control
```

Interactive alternative:
```bash
sudo ./kutrace_control
  control> init
  control> go        # (or goipc / gowrap / goipcwrap ...)
  control> wait 5
  control> stop myrun
  control> quit
```

One-shot on/off:
```bash
sudo ./kutrace_control 1    # start
sudo ./kutrace_control 0    # stop
```

## 3. Post-process to HTML timeline
```bash
./postproc3.sh smoke "kutrace smoke test"
# produces smoke.json + smoke.html
```

## 4. Unload module (optional)
```bash
sudo rmmod kutrace_mod
```

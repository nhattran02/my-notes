# Performance & system status

Checking CPU, memory, temperature, storage speed and boot time on the board.

## CPU

```bash
nproc
lscpu
cat /proc/cpuinfo | grep -E "processor|CPU part"
```

Frequency and governor (cpufreq):

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

Fix the CPU at maximum frequency for benchmarks (repeatable numbers):

```bash
for g in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo performance > $g; done
```

## Load and processes

```bash
top                         # press 1 for per-CPU view, M to sort by memory
uptime
ps aux --sort=-%cpu | head
```

Generate CPU load for thermal or stability tests (if `stress-ng` is in the image):

```bash
stress-ng --cpu $(nproc) --timeout 300s --metrics-brief
```

Without `stress-ng`, one busy loop per core:

```bash
for i in $(seq $(nproc)); do (while :; do :; done) & done
# stop:
kill $(jobs -p)
```

## Temperature

```bash
for z in /sys/class/thermal/thermal_zone*; do echo "$(cat $z/type): $(( $(cat $z/temp) / 1000 )) C"; done
```

Watch it during a load test:

```bash
watch -n 1 'cat /sys/class/thermal/thermal_zone*/temp'
```

Trip points (where throttling or shutdown happens):

```bash
grep . /sys/class/thermal/thermal_zone0/trip_point_*
```

## Memory

```bash
free -h
grep -E "MemTotal|MemAvailable|Cma" /proc/meminfo
```

`CmaTotal` / `CmaFree` is the contiguous memory used by video, GPU and DRP-AI buffers. When multimedia fails to allocate buffers, check that `CmaFree` is large enough.

Largest memory users:

```bash
ps aux --sort=-rss | head
```

## Storage speed

Write and read test on the SD card or eMMC (bypasses the page cache with `direct`):

```bash
dd if=/dev/zero of=/home/root/test.bin bs=1M count=256 oflag=direct status=progress
dd if=/home/root/test.bin of=/dev/null bs=1M iflag=direct status=progress
rm /home/root/test.bin
```

Which device and filesystem the rootfs is on:

```bash
df -h /
lsblk -f
```

## Boot time

With systemd:

```bash
systemd-analyze
systemd-analyze blame | head -20
systemd-analyze critical-chain
```

Kernel part: timestamps in the log show when each driver finished. The time of the last kernel message before init starts is the kernel boot time:

```bash
dmesg | grep -E "Run /sbin/init|Freeing unused kernel memory"
```

Add `initcall_debug` to `bootargs` to see the duration of every driver init (see [Kernel debugging](kernel-debug.md)):

```bash
dmesg | grep "initcall" | sort -t' ' -k8 -n -r | head
```

## Profiling with perf

If the image includes `perf` (`IMAGE_FEATURES += "tools-profile"`):

```bash
perf top                                  # live, system-wide
perf stat ./app                           # counters for one run
perf record -g ./app && perf report       # where the time goes, with call graphs
```

## Network throughput

See [Network & file transfer](network.md#check-the-link) (`iperf3`, `ethtool`).

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `cpufreq` directory missing | cpufreq driver not enabled, or the SoC runs at a fixed frequency in this BSP |
| Performance drops after a few minutes of load | Thermal throttling. Check temperature and `scaling_cur_freq` during the test |
| `dd` speeds unrealistically high | Page cache used. Keep `oflag=direct` / `iflag=direct`, or `sync; echo 3 > /proc/sys/vm/drop_caches` before reading |
| Video / GPU app fails with `Cannot allocate memory` while `free` shows plenty | CMA exhausted. Check `CmaFree`; increase CMA in the device tree `reserved-memory` or with `cma=` boot argument |

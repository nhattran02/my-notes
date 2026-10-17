# Kernel debugging

Finding out why a driver does not probe, a device does not work, or the kernel crashes.

Most commands need `CONFIG_DEBUG_FS=y` and `CONFIG_DYNAMIC_DEBUG=y` (see [Kernel config](../kernel/config.md)). Mount debugfs if it is not mounted already:

```bash
mount | grep -q debugfs || mount -t debugfs none /sys/kernel/debug
```

## Kernel log

```bash
dmesg -T                          # human-readable timestamps
dmesg -w                          # follow new messages
dmesg --level=err,warn            # only errors and warnings
dmesg | grep -iE "spi|1004b000"   # by driver name or register address
dmesg -C                          # clear, then reproduce the problem
```

## Boot arguments for debugging

Append to `bootargs` in U-Boot (see [Environment presets](../uboot/env-presets.md) for where `bootargs` is built):

| Argument | Effect |
| -------- | ------ |
| `loglevel=8` | Print all messages to the console, including debug |
| `ignore_loglevel` | Same, regardless of later `printk` settings |
| `initcall_debug` | Log each driver init call with its duration. Shows where boot hangs or what is slow |
| `earlycon` | Console output before the serial driver loads, using `stdout-path` from the device tree. For hangs right after `Starting kernel ...` |
| `panic=0` | Stay at the panic message instead of rebooting (`panic=1` reboots after 1 s, so the message scrolls away) |
| `clk_ignore_unused` | Do not turn off unused clocks at boot. Tests whether a missing clock reference in the device tree causes a hang |
| `dyndbg="file drivers/spi/* +p"` | Enable dynamic debug messages from boot |

Example, set temporarily for one boot:

```
setenv bootargs "${bootargs} loglevel=8 initcall_debug panic=0"
boot
```

## Dynamic debug

Turn on `dev_dbg()` / `pr_debug()` messages of one driver at runtime, without rebuilding:

```bash
echo 'file drivers/spi/spi-rspi.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module rspi +p' > /sys/kernel/debug/dynamic_debug/control       # whole module
echo 'func rspi_probe +p' > /sys/kernel/debug/dynamic_debug/control   # one function
```

Turn off again with `-p`. See what is enabled:

```bash
grep "=p" /sys/kernel/debug/dynamic_debug/control
```

For a module, enable it at load time:

```bash
modprobe rspi dyndbg=+p
```

## Why did the driver not probe?

Is the device known, and is a driver bound?

```bash
ls /sys/bus/platform/devices/ | grep 1004b000
ls -l /sys/bus/platform/devices/1004b000.spi/driver
```

Devices waiting for a resource (clock, regulator, GPIO, PHY) that is not available yet:

```bash
cat /sys/kernel/debug/devices_deferred
```

Checks in order:

1. Node `status = "okay"` in the live tree: `cat /proc/device-tree/soc/spi@1004b000/status`
2. Driver built: `zcat /proc/config.gz | grep CONFIG_SPI_RSPI`
3. Driver loaded (if module): `lsmod | grep rspi`
4. Probe error in `dmesg`, often with an error code like `-517` (`-EPROBE_DEFER`) or `-22` (`-EINVAL`)

Error codes: `-2` ENOENT, `-12` ENOMEM, `-16` EBUSY, `-19` ENODEV, `-22` EINVAL, `-110` ETIMEDOUT, `-517` EPROBE_DEFER. The full list is in `include/uapi/asm-generic/errno-base.h` and `errno.h`.

## Clocks, pins, interrupts, GPIO

```bash
cat /sys/kernel/debug/clk/clk_summary | grep -iE "spi|rspi"    # enable count, rate
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -v UNCLAIMED
cat /sys/kernel/debug/gpio
cat /proc/interrupts                                           # IRQ counts per CPU
ls /sys/kernel/debug/regmap/                                   # register dumps for regmap drivers
```

`/proc/interrupts` before and after a test shows whether the device's interrupt fires at all.

## Decode a kernel oops

An oops prints a call trace like `rspi_transfer_one+0x48/0x1a0 [rspi]`. Turn it into file and line numbers with the `vmlinux` (or `.ko`) from the **same build**, on the host:

```bash
cd $KSRC
./scripts/faddr2line vmlinux rspi_transfer_one+0x48/0x1a0
./scripts/faddr2line drivers/spi/spi-rspi.o rspi_transfer_one+0x48/0x1a0    # for a module
```

Decode a whole saved log:

```bash
./scripts/decode_stacktrace.sh vmlinux $KSRC < oops.txt
```

!!! tip
    Build with `CONFIG_DEBUG_INFO=y` to get line numbers. Without it, only function names are available. In Yocto, `vmlinux` is in `tmp/work/*/linux-renesas/*/build/`.

## ftrace

Trace function calls of a driver, without adding prints:

```bash
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function > current_tracer
echo 'rspi_*' > set_ftrace_filter
echo 1 > tracing_on
# reproduce the problem
echo 0 > tracing_on
cat trace | head -50
```

Call graph with timing:

```bash
echo function_graph > current_tracer
```

Trace events, e.g. all SPI events or all IRQ handlers:

```bash
echo 1 > events/spi/enable
echo 1 > events/irq/irq_handler_entry/enable
```

Reset:

```bash
echo nop > current_tracer
echo > set_ftrace_filter
echo 0 > events/enable
```

Requires `CONFIG_FTRACE=y`, `CONFIG_FUNCTION_TRACER=y`, `CONFIG_FUNCTION_GRAPH_TRACER=y`.

## Magic SysRq

When the system is stuck but the console still responds (`CONFIG_MAGIC_SYSRQ=y`):

```bash
echo 1 > /proc/sys/kernel/sysrq
echo t > /proc/sysrq-trigger      # stack traces of all tasks
echo w > /proc/sysrq-trigger      # blocked (D state) tasks only
echo m > /proc/sysrq-trigger      # memory info
echo b > /proc/sysrq-trigger      # reboot immediately
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| Nothing after `Starting kernel ...` | Wrong dtb, wrong load address, or console not set. Add `earlycon`; check `console=ttySC0,115200n8` |
| Boot hangs at a random point, no message | Often a clock or power domain turned off. Try `clk_ignore_unused`; use `initcall_debug` to see the last init call |
| `/sys/kernel/debug` is empty | debugfs not mounted (see top of page) or `CONFIG_DEBUG_FS` off |
| `dynamic_debug/control` missing | `CONFIG_DYNAMIC_DEBUG` off. Add `#define DEBUG` at the top of the driver file as a fallback |
| `faddr2line` prints `??` | `vmlinux` from a different build, or no debug info |

# Peripherals from user space

Quick tests of GPIO, I2C, SPI, UART, CAN, ADC, PWM and raw registers from the board's shell, without writing any code.

Tools needed in the image: `libgpiod-tools i2c-tools spidev-test can-utils devmem2` (see [BitBake commands](../yocto/bitbake.md) for adding packages).

## GPIO (libgpiod)

```bash
gpiodetect
gpioinfo gpiochip0 | head -40
```

On RZ/G2L-family SoCs, the line number of port pin `Pm_n` is `m * 8 + n` (same as `RZG2L_GPIO(m, n)` in the device tree). Example: `P43_1` → line `345`.

```bash
echo $((43 * 8 + 1))
```

=== "libgpiod v1 (Dunfell, Kirkstone)"

    ```bash
    gpioset gpiochip0 345=1           # drive high
    gpioset gpiochip0 345=0           # drive low
    gpioget gpiochip0 345             # read input
    gpiomon gpiochip0 345             # wait for edges (Ctrl+C to stop)
    ```

=== "libgpiod v2 (Scarthgap and later)"

    ```bash
    gpioset -c gpiochip0 345=1        # keeps running and holds the value; Ctrl+C to release
    gpioget -c gpiochip0 345
    gpiomon -c gpiochip0 345
    ```

Check the version with `gpioset --version`.

!!! note
    `gpioset` fails with `Device or resource busy` if the pin is used by a driver or as a peripheral function. `gpioinfo` shows the consumer in the `[used]` column.

LEDs defined with `gpio-leds` in the device tree are controlled through sysfs instead:

```bash
ls /sys/class/leds/
echo 1 > /sys/class/leds/led-user/brightness
echo heartbeat > /sys/class/leds/led-user/trigger
```

## I2C

```bash
i2cdetect -l                      # list buses
i2cdetect -y 1                    # scan bus 1 (UU = address used by a kernel driver)
```

Read / write one register (bus 1, device 0x50, register 0x00):

```bash
i2cget -y 1 0x50 0x00
i2cset -y 1 0x50 0x00 0xab
i2cdump -y 1 0x50                 # all registers 0x00-0xff
```

Multi-byte transfer (write register address 0x00, then read 8 bytes):

```bash
i2ctransfer -y 1 w1@0x50 0x00 r8
```

!!! warning
    For addresses shown as `UU`, a driver owns the device. `i2cset` needs `-f` then, and can confuse the driver. Prefer reading.

## SPI (spidev)

`/dev/spidevX.Y` exists only when a device on that bus is bound to the `spidev` driver:

```bash
ls /dev/spidev*
```

If there is none, bind one at runtime (the bus controller must be enabled in the device tree with a child node, e.g. `spi1.0`):

```bash
echo spidev > /sys/bus/spi/devices/spi1.0/driver_override
echo spi1.0 > /sys/bus/spi/drivers/spidev/bind
```

Loopback test: connect MOSI to MISO, then:

```bash
spidev_test -D /dev/spidev1.0 -s 1000000 -v -p "\x01\x02\x03\x04"
```

The `RX` line must equal the `TX` line. All `00` or `FF` means no loopback or wrong pins.

## UART

Ports other than the console:

```bash
ls /dev/ttySC*
stty -F /dev/ttySC1 115200 raw -echo
```

Loopback test (connect TX to RX):

```bash
cat /dev/ttySC1 &
echo "hello uart" > /dev/ttySC1
kill %1
```

Interactive (if `picocom` or `microcom` is in the image):

```bash
microcom -s 115200 /dev/ttySC1
```

## CAN / CAN FD

```bash
ip link set can0 down
ip link set can0 type can bitrate 500000
ip link set can0 up
ip -details -statistics link show can0
```

CAN FD (nominal 500 kbit/s, data 2 Mbit/s):

```bash
ip link set can0 type can bitrate 500000 dbitrate 2000000 fd on
```

Test between two channels wired together (e.g. `can0` ↔ `can1` with a terminated cable):

```bash
ip link set can1 type can bitrate 500000 && ip link set can1 up
candump can1 &
cansend can0 123#DEADBEEF
cansend can0 123##1.0011223344556677       # CAN FD frame
kill %1
```

## ADC (IIO)

```bash
ls /sys/bus/iio/devices/
cat /sys/bus/iio/devices/iio:device0/name
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw
```

## PWM

```bash
ls /sys/class/pwm/
echo 0 > /sys/class/pwm/pwmchip0/export
echo 1000000 > /sys/class/pwm/pwmchip0/pwm0/period         # ns -> 1 kHz
echo 500000  > /sys/class/pwm/pwmchip0/pwm0/duty_cycle     # 50 %
echo 1 > /sys/class/pwm/pwmchip0/pwm0/enable
```

## Raw registers (devmem)

Read a register directly, e.g. to check a pin function or a clock enable bit. Needs `CONFIG_DEVMEM=y`.

```bash
devmem2 0x11030000 w                 # read 32-bit word
devmem2 0x11030000 w 0x00000001      # write
```

BusyBox has an equivalent built in:

```bash
devmem 0x11030000 32
```

!!! danger
    Writing a wrong register can hang the board or damage connected hardware. Read first, change one bit at a time.

!!! note
    Reading registers of a peripheral whose clock is off returns `0` or hangs the bus. Make sure the driver is loaded (clock enabled) before reading its registers.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `gpioset: error setting the GPIO line values: Device or resource busy` | Pin claimed by a driver or used as a peripheral function. Check `gpioinfo` and the pinmux in `/sys/kernel/debug/pinctrl/` |
| `i2cdetect` shows nothing | Bus not enabled in DT, wrong bus number, missing pull-ups, or device not powered |
| `Error: Could not open file /dev/i2c-1` | `CONFIG_I2C_CHARDEV` not enabled, or the bus is not enabled in the device tree |
| `can't open device: No such file or directory` for spidev | No device bound to `spidev`. Use `driver_override` as above |
| `RTNETLINK answers: Operation not supported` on CAN FD | Controller in classical CAN mode, or kernel without CAN FD support for this controller |
| `devmem2: Bus error` | Address not mapped, or peripheral clock is off |

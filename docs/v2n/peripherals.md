# Peripherals

## MIPI CSI-2 camera + ISP

RZ/V2N routes the camera through a hardware ISP before the capture unit:

```
Sensor → CSI-2 → ISP → CRU → /dev/videoN
```

```bash
media-ctl -p                       # inspect and check link states
v4l2-ctl --list-devices
media-ctl -V '"sensor":0 [fmt:SRGGB10/1920x1080]'

# preview
gst-launch-1.0 v4l2src device=/dev/video0 ! \
    video/x-raw,width=1920,height=1080 ! waylandsink
```

If the ISP node is missing, confirm the `isp` machine feature was enabled in the
Yocto build and that the ISP firmware blob is in `/lib/firmware`.

## Camera → DRP-AI pipeline

Typical demo flow: `v4l2src → ISP → format convert → DRP-AI inference → overlay →
waylandsink`. The BSP ships sample apps that wire this together.

## Display

```bash
modetest -M rzv2n-du -c
weston --backend=drm-backend.so
```

## Networking

- `eth0` — GbE. MAC from on-board EEPROM; set `ethaddr` in U-Boot if blank.
- USB 3.2 host ports for storage / peripherals.

## GPIO / I2C / SPI

```bash
gpioinfo
gpioset gpiochip0 5=1
i2cdetect -y 0
spidev_test -D /dev/spidev0.0 -s 1000000
```

## Cortex-M33 / remoteproc

```bash
echo rzv2n_cm33_fw.elf > /sys/class/remoteproc/remoteproc0/firmware
echo start > /sys/class/remoteproc/remoteproc0/state
cat /sys/kernel/debug/remoteproc/remoteproc0/trace0
```

Keep the CM33 SRAM/DDR carveout in the device tree consistent with the CM33
linker script.

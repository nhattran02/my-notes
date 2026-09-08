# Peripherals

## MIPI CSI-2 camera

The EVK ships with an OV5645 module on the CSI connector.

```bash
media-ctl -p                       # inspect the pipeline
v4l2-ctl --list-devices
# capture one frame
gst-launch-1.0 v4l2src device=/dev/video0 num-buffers=1 ! \
    video/x-raw,width=1280,height=960 ! jpegenc ! filesink location=frame.jpg
```

Pipeline: `OV5645 → CRU (csi2) → /dev/videoN`.

## Display (DSI / HDMI via DSI-to-HDMI)

```bash
modetest -M rzg2l-du -c            # list connectors
weston --backend=drm-backend.so
```

## Networking

- `eth0` / `eth1` — GbE via the RZ/G2L GbE controllers (`ravb`-style driver).
- MAC address is read from the on-board EEPROM; override with `ethaddr` in U-Boot
  if blank.

## CAN

```dts
&canfd {
    pinctrl-0 = <&can0_pins &can1_pins>;
    pinctrl-names = "default";
    status = "okay";
    channel0 { status = "okay"; };
};
```

```bash
ip link set can0 type can bitrate 500000
ip link set can0 up
cansend can0 123#DEADBEEF
candump can0
```

## I2C / SPI / GPIO

```bash
i2cdetect -y 0
spidev_test -D /dev/spidev1.0 -s 1000000
gpioinfo
gpioset gpiochip0 12=1
```

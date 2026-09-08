# Board setup

## Connections

1. Seat the SMARC SoM on the carrier and latch both clips.
2. Connect the **USB-micro (CN14)** debug port to the host — enumerates as
   `/dev/ttyUSB0` and `/dev/ttyUSB1` (use the first one for the console).
3. Insert a prepared microSD card (or plan to boot from eMMC / QSPI).
4. Apply 5 V / 3 A on the barrel jack. `SW23` is the power switch.

## Serial console

```bash
sudo picocom -b 115200 /dev/ttyUSB0
# or
sudo minicom -D /dev/ttyUSB0 -b 115200
```

Console settings: **115200 8N1**, no flow control.

## Writing an SD card image

```bash
xz -dk core-image-weston-smarc-rzg2l.wic.xz
sudo dd if=core-image-weston-smarc-rzg2l.wic of=/dev/sdX bs=4M conv=fsync status=progress
```

!!! warning
    Double-check `/dev/sdX` with `lsblk` before running `dd`.

## Network

- Two 1000BASE-T ports (`eth0`, `eth1`).
- Default images bring up `eth0` via DHCP.
- For a fixed address during development:

```bash
ip addr add 192.168.10.20/24 dev eth0
ip link set eth0 up
```

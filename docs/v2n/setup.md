# Board setup

## Connections

1. Connect the debug **USB-C / USB-micro** port to the host — enumerates as
   `/dev/ttyUSB0` (console) and a second port.
2. Insert a prepared microSD card, or plan to boot from eMMC / QSPI.
3. Connect Ethernet and, if needed, the MIPI camera / display.
4. Apply the supplied USB-PD or barrel-jack power and toggle the power switch.

## Serial console

```bash
sudo picocom -b 115200 /dev/ttyUSB0
```

Console settings: **115200 8N1**, no flow control.

## Writing an SD card image

```bash
xz -dk core-image-weston-rzv2n-evk.wic.xz
sudo dd if=core-image-weston-rzv2n-evk.wic of=/dev/sdX bs=4M conv=fsync status=progress
sync
```

!!! warning
    Confirm the target device with `lsblk` before `dd`.

## Network

```bash
ip addr show eth0
# static during bring-up
ip addr add 192.168.10.30/24 dev eth0
ip link set eth0 up
```

## First boot checklist

- [ ] U-Boot prompt reachable over serial
- [ ] Kernel boots to a login prompt
- [ ] `dmesg | grep -i drp` shows the DRP-AI device probing
- [ ] `v4l2-ctl --list-devices` lists the CRU / ISP video nodes

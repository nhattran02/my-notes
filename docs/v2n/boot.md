# Boot modes & flashing

## Boot mode switches

The EVK DIP switches select the boot source — check the RZ/V2N EVK user manual
for the exact pattern of your board revision:

| Mode          | Use                                       |
| ------------- | ----------------------------------------- |
| eSD           | boot from microSD                         |
| eMMC          | boot from on-board eMMC                   |
| QSPI          | boot from serial NOR                      |
| SCIF download | UART boot — used to flash the bootloaders |

## Boot chain

```
BootROM → bl2 (TF-A) → fip (bl31 + U-Boot) → Linux kernel + DTB
```

## Flashing the bootloader (SCIF download mode)

1. Set the switches to **SCIF download**, power-cycle.
2. Send the `Flash_Writer_SCIF_*.mot` binary over the serial terminal.
3. At the Flash Writer prompt use `XLS2` to write the bl2 and fip images to the
   addresses listed in the release notes for your BSP version
   (they differ per BSP release, so follow the release `.txt`).
4. Set the switches back to **QSPI** (or eMMC) and power-cycle.

## Writing images to eMMC from U-Boot

```text
# load into RAM (from SD / TFTP), then:
mmc dev 0 1
mmc write 0x48000000 0 <blocks>
```

## U-Boot environment

```text
setenv bootargs 'console=ttySC0,115200 root=/dev/mmcblk1p2 rw rootwait'
setenv bootcmd 'mmc dev 1; ext4load mmc 1:2 0x48080000 /boot/Image; ext4load mmc 1:2 0x48000000 /boot/r9a09g056n48-rzv2n-evk.dtb; booti 0x48080000 - 0x48000000'
saveenv
```

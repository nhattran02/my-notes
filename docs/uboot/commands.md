# U-Boot commands

Commands I use at the U-Boot prompt (`=>`). Stop autoboot by pressing any key during `Hit any key to stop autoboot`.

Load addresses below are the ones used on RZ/G2L (DDR starts at `0x40000000`): kernel `0x48080000`, dtb `0x48000000`.

## Help and info

```
help
help mmc
version
bdinfo
```

## Environment

```
printenv
printenv bootcmd bootargs
setenv myvar 'value with spaces'
setenv myvar                         # delete a variable
editenv bootargs                     # edit in place (no retyping)
saveenv
env default -a                       # reset everything to defaults (then saveenv)
```

Run a script stored in a variable:

```
run nfsboot
```

Set `bootdelay` (seconds before autoboot; `-1` = never autoboot, `0` = no delay but still interruptible by keypress if enabled):

```
setenv bootdelay 3
saveenv
```

## Network

```
setenv ipaddr 192.168.1.5
setenv serverip 192.168.1.3
ping ${serverip}
tftp 0x48080000 Image
dhcp                                  # get IP from a DHCP server instead
```

After `tftp`, `${filesize}` holds the number of bytes loaded (hex). Useful for writing the file somewhere else.

## MMC (SD card / eMMC)

On RZ/G2L SMARC: `mmc 0` = eMMC, `mmc 1` = micro SD.

```
mmc list
mmc dev 1
mmc info
mmc part
mmc rescan                            # after inserting a card
```

## Filesystems

ext4:

```
ext4ls mmc 1:1
ext4ls mmc 1:2 /boot
ext4load mmc 1:1 0x48080000 Image
```

FAT:

```
fatls mmc 1:1
fatload mmc 1:1 0x48080000 Image
```

Generic (filesystem type detected automatically):

```
ls mmc 1:1
load mmc 1:1 0x48080000 Image
```

## USB storage

```
usb start
usb storage
ls usb 0:1
load usb 0:1 0x48080000 Image
```

## Memory

Display (`md`), write (`mw`), copy (`cp`), compare (`cmp`):

```
md.l 0x48000000 0x20                  # 0x20 words of 32 bit
mw.l 0x48000000 0xdeadbeef 1
cp.b 0x48080000 0x50000000 ${filesize}
cmp.b 0x48080000 0x50000000 ${filesize}
```

Peek a peripheral register (read-only check, e.g. a pin function register):

```
md.l <register-address> 1
```

!!! warning
    Writing peripheral registers with `mw` can hang the board. Prefer `md` for checks.

## Device tree in RAM

After loading the dtb, inspect or modify it before booting:

```
fdt addr 0x48000000
fdt print /chosen
fdt print /soc/serial@1004b800
fdt list /
fdt set /soc/spi@1004b000 status "okay"
fdt resize 8192                       # make room before adding nodes or applying overlays
```

## Boot

```
booti 0x48080000 - 0x48000000         # arm64 Image, no initramfs, dtb
booti 0x48080000 0x49000000:${filesize} 0x48000000   # with initramfs
boot                                  # run bootcmd
reset
```

## Scripting

Conditionals, useful inside stored scripts:

```
if ping ${serverip}; then run nfsboot; else run sdboot; fi
```

```
if test -e mmc 1:1 Image; then echo "Image found on SD"; fi
```

## QSPI flash

The bootloaders are in QSPI. Read only; writing is done with Flash Writer ([Flashing the bootloader](../g2l/001boot.md)):

```
sf probe
sf read 0x48000000 0x0 0x1000
md.b 0x48000000 0x40
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `Unknown command 'ext4load'` | Command not built into this U-Boot. Try the generic `load` |
| `** Bad device specification mmc 1 **` | No card, or wrong device number. `mmc list`, `mmc rescan` |
| `TFTP error: trying to overwrite reserved memory` | Load address overlaps U-Boot itself. Use the standard addresses above |
| `Bad Linux ARM64 Image magic!` | File loaded is not an arm64 `Image` (wrong file, or a compressed `Image.gz`), or loaded to a different address than `booti` uses |
| `ERROR: Did not find a cmdline Flattened Device Tree` | dtb not loaded, or loaded to another address than given to `booti` |

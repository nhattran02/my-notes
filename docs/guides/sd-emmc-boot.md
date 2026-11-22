# Booting from SD card and eMMC

Instead of loading the kernel over the network every time, `Image`, the device tree and the root filesystem can be stored on a micro SD card or on the on-module eMMC. U-Boot (in SPI flash, see [Flashing the bootloader](flash-bootloader.md)) then loads them with `ext4load`.

Both media use the same layout:

| Partition | Size      | Filesystem | Content                     |
| --------- | --------- | ---------- | --------------------------- |
| 1         | 1 GB      | ext4       | `Image`, `<board>.dtb`      |
| 2         | remaining | ext4       | Root filesystem             |

The procedure is the same on every board. What changes is the device tree name and how Linux and U-Boot number the MMC devices:

=== "RZ/G2L"

    | Medium                   | Linux device   | U-Boot device |
    | ------------------------ | -------------- | ------------- |
    | eMMC (on SoM)            | `/dev/mmcblk0` | `mmc 0`       |
    | micro SD (carrier board) | `/dev/mmcblk1` | `mmc 1`       |

    Device tree: `r9a07g044l2-smarc.dtb`

=== "RZ/G3S"

    U-Boot lists 3 SD hosts: `sd@11c00000: 0, sd@11c10000: 1, sd@11c20000: 2`.

    - SD0 (`mmc 0`) is connected either to the eMMC or to the micro SD slot uSD0, selected by **SW_CONFIG[2]** on the module (OFF = eMMC, ON = uSD0).
    - With the default settings, the Start-up Guide boots Linux from the micro SD card on `mmc 1`.

    Check the Linux device names with `lsblk` before partitioning.

    Device tree: `r9a08g045s33-smarc.dtb`

=== "RZ/V2N"

    | Medium | Linux device   | U-Boot device |
    | ------ | -------------- | ------------- |
    | eMMC   | `/dev/mmcblk0` | not checked   |

    Check the micro SD device name with `lsblk`.

    Device tree: `r9a09g056n48-rzv2n-evk.dtb`

!!! note
    The steps below are run **on the board**, after booting Linux with [TFTP and NFS](nfs-boot.md). This is the only way to write to the eMMC. Copy `Image`, the device tree and the rootfs tarball into the NFS rootfs (for example to `/nfs/<board>/home/root/`) so they are available on the board.

    An SD card can also be prepared on a Linux host with a card reader. In that case the card shows up as `/dev/sdX` (check with `lsblk`) instead of `/dev/mmcblkN`.

The commands below use the RZ/G2L SD card (`/dev/mmcblk1`). Replace the device with the one of your board and medium.

## Create partitions

!!! danger
    Partitioning erases all data on the device. Double-check the device name with `lsblk` before continuing.

Unmount any partitions that were mounted automatically, then open `fdisk`:

```bash
umount /dev/mmcblk1p* 2>/dev/null
fdisk /dev/mmcblk1
```

Create partition 1 (1 GB) for the kernel and device tree:

```
Command (m for help): o                                # new empty DOS partition table
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): <Enter>
Partition number (1-4, default 1): <Enter>
First sector (2048-62333951, default 2048): <Enter>
Last sector, +/-sectors or +/-size{K,M,G,T,P} (...): +1G
```

Create partition 2 using the remaining space for the root filesystem:

```
Command (m for help): n
Select (default p): <Enter>
Partition number (2-4, default 2): <Enter>
First sector (...): <Enter>
Last sector, +/-sectors or +/-size{K,M,G,T,P} (...): <Enter>
```

Write the partition table and exit:

```
Command (m for help): w
```

## Format partitions

```bash
mkfs.ext4 -L boot   /dev/mmcblk1p1
mkfs.ext4 -L rootfs /dev/mmcblk1p2
```

## Copy kernel, device tree and rootfs

Copy `Image` and the device tree to partition 1:

```bash
mount /dev/mmcblk1p1 /mnt
cp ~/Image ~/<board>.dtb /mnt/
sync
umount /mnt
```

Extract the root filesystem to partition 2:

```bash
mount /dev/mmcblk1p2 /mnt
tar -xf ~/<rootfs>.tar.bz2 -C /mnt
sync
umount /mnt
```

!!! tip
    Extract the tarball directly into the mount point with `tar -C` rather than copying the archive onto the partition first. This avoids leaving a large `.tar.bz2` file in the root filesystem.

## U-Boot config

Power on the board and press any key to stop autoboot.

The general form is:

```
setenv bootcmd 'mmc dev <N>; ext4load mmc <N>:1 <kernel addr> Image; ext4load mmc <N>:1 <fdt addr> <board>.dtb; booti <kernel addr> - <fdt addr>'
setenv bootargs 'console=<console> rootwait root=/dev/mmcblk<M>p2 rw'
```

- `<N>`: U-Boot MMC device number, `<M>`: Linux MMC device number (see the table at the top).
- `<kernel addr>` / `<fdt addr>`: the same load addresses as in the [TFTP setup](nfs-boot.md#u-boot-config-on-the-board).

=== "RZ/G2L"

    SD card:

    ```
    setenv bootcmd 'mmc dev 1; ext4load mmc 1:1 0x48080000 Image; ext4load mmc 1:1 0x48000000 r9a07g044l2-smarc.dtb; booti 0x48080000 - 0x48000000'
    setenv bootargs 'console=ttySC0,115200n8 rootwait root=/dev/mmcblk1p2 rw noinitrd nohlt panic=1 earlyprintk=ttySC0,115200n8'
    ```

    eMMC:

    ```
    setenv bootcmd 'mmc dev 0; ext4load mmc 0:1 0x48080000 Image; ext4load mmc 0:1 0x48000000 r9a07g044l2-smarc.dtb; booti 0x48080000 - 0x48000000'
    setenv bootargs 'console=ttySC0,115200n8 rootwait root=/dev/mmcblk0p2 rw noinitrd nohlt panic=1 earlyprintk=ttySC0,115200n8'
    ```

=== "RZ/G3S"

    Not tested yet. Fill in the general form with `r9a08g045s33-smarc.dtb`, the MMC numbers above and the load addresses `0x48080000` (kernel) / `0x48000000` (fdt).

=== "RZ/V2N"

    Not tested yet. Run `printenv bootcmd bootargs` to see the default load addresses and console, then fill in the general form.

What the commands do:

- `mmc dev <N>`: select the MMC device.
- `ext4load mmc <dev>:<part> <addr> <file>`: load a file from an ext4 partition into RAM.
- `booti <kernel> - <fdt>`: boot an arm64 `Image`. The `-` means there is no initramfs.
- `root=/dev/mmcblkXp2 rootwait`: mount partition 2 as rootfs, and wait until the MMC device is ready.

!!! warning
    The file names in `ext4load` must match the files on partition 1 exactly. If you stored them under other names (for example `Image_scif`), update `bootcmd` accordingly.

Before saving, you can check that U-Boot sees the files:

```
mmc dev 1
ext4ls mmc 1:1
```

Save the environment so the board boots this way automatically (optional), then boot:

```
saveenv
boot
```

## Network access after boot (optional)

Unlike NFS boot, the kernel does not configure Ethernet when booting from SD/eMMC. To reach the board from the host (for `ssh`, `scp`, ...), bring up `eth0` with a static IP on the same subnet as the host:

```bash
ip link set eth0 up
ip addr add 192.168.1.5/24 dev eth0
ip addr show eth0
ping 192.168.1.3      # host IP
```

The host can then connect with `ssh root@192.168.1.5`. This setting is lost on reboot.

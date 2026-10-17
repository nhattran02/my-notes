# Build output

Where the build results are, which file is used for what, and how to copy them to the TFTP/NFS server or an SD card.

## Variables

```bash
export BSP_DIR=~/yocto/rzg2l_vlp_v3.0.x
export MACHINE=smarc-rzg2l
export IMAGE=core-image-minimal
export DEPLOY=$BSP_DIR/build/tmp/deploy/images/$MACHINE
```

## Files

```bash
ls -l $DEPLOY
```

Files for RZ/G2L SMARC (other boards follow the same pattern with their own names):

| File | Used for |
| ---- | -------- |
| `Image` | Kernel. Loaded by U-Boot (TFTP, SD, eMMC) |
| `r9a07g044l2-smarc.dtb` | Device tree for the board |
| `modules-$MACHINE.tgz` | Kernel modules (`/lib/modules/...`), already included in the rootfs |
| `$IMAGE-$MACHINE.tar.bz2` | Root filesystem as a tarball. Extract to NFS export or an SD/eMMC partition |
| `$IMAGE-$MACHINE.ext4` | Root filesystem as an ext4 image |
| `$IMAGE-$MACHINE.wic.*` | Full disk image (partitions + kernel + rootfs). Write directly to an SD card |
| `$IMAGE-$MACHINE.wic.bmap` | Block map for fast writing with `bmaptool` |
| `Flash_Writer_SCIF_*.mot` | Flash Writer, sent over SCIF in SCIF download mode |
| `bl2_bp-*.srec`, `fip-*.srec` | Bootloaders (TF-A BL2, FIP = BL31 + U-Boot) for writing to QSPI |
| `bl2_bp-*.bin`, `fip-*.bin` | Same bootloaders in binary form (eMMC boot partitions) |

Which bootloader files to use is described in [Flashing the bootloader](../g2l/001boot.md).

!!! note
    The compressed format of `.wic` (`.wic.gz`, `.wic.xz`, `.wic.bz2`) depends on `IMAGE_FSTYPES` in the BSP. Check with:

    ```bash
    bitbake -e $IMAGE | grep ^IMAGE_FSTYPES=
    ```

## Symlinks vs. timestamped files

Each build creates timestamped files (for example `core-image-minimal-smarc-rzg2l-20261017081530.rootfs.tar.bz2`) plus a stable symlink pointing to the latest one. Always use the symlink names, and copy with `-L` so the real file is copied, not the link:

```bash
cp -L $DEPLOY/Image /some/where/
```

Old timestamped files pile up over many builds. Remove ones older than 7 days:

```bash
find $DEPLOY -maxdepth 1 -type f -name "*-20[0-9]*" -mtime +7 -delete
```

## Copy to TFTP / NFS server

Matches the setup in [Booting with TFTP and NFS](../g2l/002nfs.md):

```bash
sudo cp -L $DEPLOY/Image $DEPLOY/r9a07g044l2-smarc.dtb /srv/tftp/
```

Replace the NFS rootfs with the new one:

```bash
sudo rm -rf /nfs/rzg2l/*
sudo tar -xf $DEPLOY/$IMAGE-$MACHINE.tar.bz2 -C /nfs/rzg2l
```

!!! warning
    Double-check the path in `rm -rf`. An empty or wrong variable here can delete the host's files.

## Copy to an SD card on the host

Write the full `.wic` image (all partitions at once):

```bash
lsblk                                   # find the SD card, e.g. /dev/sdb
export SDCARD=/dev/sdX
sudo umount ${SDCARD}* 2>/dev/null
sudo bmaptool copy $DEPLOY/$IMAGE-$MACHINE.wic.gz $SDCARD
```

Without `bmaptool`, decompress and use `dd` (slower, writes empty blocks too):

```bash
gunzip -c $DEPLOY/$IMAGE-$MACHINE.wic.gz | sudo dd of=$SDCARD bs=4M conv=fsync status=progress
```

To build my own partition layout instead, see [Booting from SD Card and eMMC](../g2l/003sd_emmc.md).

## SDK output

`bitbake $IMAGE -c populate_sdk` writes the SDK installer to a different directory:

```bash
ls $BSP_DIR/build/tmp/deploy/sdk/
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| Copied `Image` is only a few bytes | A symlink was copied. Use `cp -L` |
| Board still runs the old kernel | Copied to the wrong TFTP directory, or U-Boot loads from SD/eMMC. Check `printenv bootcmd` |
| No `.wic` file | `wic` is not in `IMAGE_FSTYPES`. Add `IMAGE_FSTYPES:append = " wic.gz wic.bmap"` (Dunfell syntax: `IMAGE_FSTYPES_append`) |

# Fast edit–test loop

The loop I use when changing the kernel or device tree: build on the host, copy to TFTP/NFS, reboot the board. No SD card swapping, no BitBake. One iteration takes well under a minute for a device tree change.

Prerequisites:

- Board boots with TFTP + NFS ([Booting with TFTP and NFS](../g2l/002nfs.md)), with `saveenv` done so it autoboots that way.
- Kernel tree and toolchain set up ([Build the kernel](build.md)).

## Variables

```bash
export KSRC=~/work/rz_linux-cip
export SDK_DIR=/opt/poky/3.1.x
export DTB=renesas/r9a07g044l2-smarc.dtb
export TFTP_DIR=/srv/tftp
export NFS_ROOT=/nfs/rzg2l
export BOARD_IP=192.168.1.5
```

## Deploy script

Save as `~/bin/kdeploy` and `chmod +x ~/bin/kdeploy`. It builds what is needed and copies the results to TFTP/NFS.

```bash
#!/usr/bin/env bash
# usage: kdeploy [all|kernel|dtb|modules]   (default: all)
set -euo pipefail

: "${KSRC:?set KSRC}" "${SDK_DIR:?set SDK_DIR}" "${DTB:?set DTB}"
TFTP_DIR=${TFTP_DIR:-/srv/tftp}
NFS_ROOT=${NFS_ROOT:-/nfs/rzg2l}
what=${1:-all}

source "$SDK_DIR"/environment-setup-aarch64-poky-linux
unset LDFLAGS
cd "$KSRC"

if [[ $what == all || $what == kernel ]]; then
    make -j"$(nproc)" Image
    sudo cp arch/arm64/boot/Image "$TFTP_DIR"/
fi

if [[ $what == all || $what == dtb ]]; then
    make "$DTB"
    sudo cp arch/arm64/boot/dts/"$DTB" "$TFTP_DIR"/
fi

if [[ $what == all || $what == modules ]]; then
    make -j"$(nproc)" modules
    rm -rf /tmp/kmod
    make modules_install INSTALL_MOD_PATH=/tmp/kmod INSTALL_MOD_STRIP=1
    sudo rm -rf "$NFS_ROOT"/lib/modules/"$(make -s kernelrelease)"
    sudo cp -r /tmp/kmod/lib/modules/* "$NFS_ROOT"/lib/modules/
fi

echo "Deployed ($what): $(make -s kernelrelease)"
```

## Use it

Device tree change only:

```bash
kdeploy dtb
```

Driver change built into the kernel:

```bash
kdeploy kernel
```

Then reboot the board (from its serial console or over SSH):

```bash
ssh root@$BOARD_IP reboot
```

U-Boot loads the new files from TFTP automatically.

## Faster: reload a module without rebooting

When the driver is built as a module (`=m`), there is no need to reboot. The NFS rootfs already has the new `.ko` after `kdeploy modules`:

```bash
ssh root@$BOARD_IP "rmmod my_driver; modprobe my_driver; dmesg | tail -20"
```

## Check the board runs the new build

Each build increments the build number in `uname -v` (`#2`, `#3`, ...) and updates the build time:

```bash
ssh root@$BOARD_IP uname -v
```

```
#7 SMP PREEMPT Fri Oct 17 10:12:44 +07 2026
```

For device tree changes, read the property back from the live tree:

```bash
ssh root@$BOARD_IP "cat /proc/device-tree/model; echo"
```

## Speed up rebuilds with ccache

```bash
sudo apt-get install -y ccache
make -j$(nproc) CROSS_COMPILE="ccache aarch64-poky-linux-" Image
ccache -s          # hit statistics
```

Most useful after `make clean`, switching branches, or rebuilding the same commit in several trees.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| Board boots the old kernel | U-Boot env not saved for TFTP boot (`printenv bootcmd`), or files copied to another directory than the TFTP server root |
| `TFTP error: 'File not found'` | File name in `bootcmd` differs from the copied file. The DTB is copied without the `renesas/` prefix |
| Board hangs at NFS mount after reboot | Host IP changed or NFS server stopped. Check `showmount -e` on the host |
| `modprobe: FATAL: Module not found` | `uname -r` on the board differs from the installed modules directory. Run `kdeploy all` so kernel and modules match |

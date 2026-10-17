# Build the kernel

Building the kernel from a git tree, outside Yocto. Much faster than `bitbake` for edit–build–test cycles.

## Variables

```bash
export KSRC=~/work/rz_linux-cip                 # kernel tree
export SDK_DIR=/opt/poky/3.1.x                  # installed Yocto SDK
export DTB=renesas/r9a07g044l2-smarc.dtb        # see table below
export JOBS=$(nproc)
```

| Board | `DTB` |
| ----- | ----- |
| RZ/G2L SMARC EVK | `renesas/r9a07g044l2-smarc.dtb` |
| RZ/G3S SMARC EVK | `renesas/r9a08g045s33-smarc.dtb` |
| RZ/G3E SMARC EVK | `renesas/r9a09g047e57-smarc.dtb` |
| RZ/V2N EVK | `renesas/r9a09g056n48-rzv2n-evk.dtb` |

## Get the same source as Yocto

The BSP kernel lives in [renesas-rz/rz_linux-cip](https://github.com/renesas-rz/rz_linux-cip). To work on exactly the version the BSP builds, read the branch and commit from the recipe:

```bash
cd $BSP_DIR && source poky/oe-init-build-env build
bitbake -e linux-renesas | grep -E "^(KBRANCH|SRCREV|SRC_URI)="
```

Clone and check out that commit:

```bash
git clone https://github.com/renesas-rz/rz_linux-cip.git $KSRC
cd $KSRC
git checkout -b work <SRCREV>
```

!!! note
    Patches added by `.bbappend` files in `meta-renesas` or my layer are **not** in this tree. Apply them with `git am` if I need them (see `bitbake-layers show-appends | grep linux-renesas`).

## Set up the toolchain

=== "Yocto SDK (recommended)"

    ```bash
    source $SDK_DIR/environment-setup-aarch64-poky-linux
    unset LDFLAGS
    ```

    The SDK sets `ARCH=arm64` and `CROSS_COMPILE=aarch64-poky-linux-`. `unset LDFLAGS` is required: the SDK's value is meant for gcc and breaks the kernel link.

=== "Ubuntu cross compiler"

    ```bash
    sudo apt-get install -y gcc-aarch64-linux-gnu bison flex libssl-dev bc
    export ARCH=arm64
    export CROSS_COMPILE=aarch64-linux-gnu-
    ```

## Configure

Option 1: the default config of the tree:

```bash
cd $KSRC
make defconfig
```

Option 2: the **exact** config Yocto used (best when replacing the kernel of an existing image):

```bash
cp $BSP_DIR/build/tmp/work/*/linux-renesas/*/build/.config $KSRC/.config
make olddefconfig
```

## Build

Everything (kernel, all device trees, modules):

```bash
make -j$JOBS Image dtbs modules
```

Only the kernel, or only one device tree:

```bash
make -j$JOBS Image
make $DTB
```

Output:

| File | Path |
| ---- | ---- |
| Kernel | `arch/arm64/boot/Image` |
| Device tree | `arch/arm64/boot/dts/$DTB` |
| Kernel release string | `make -s kernelrelease` |

## Install

Kernel and dtb to the TFTP server (see [Booting with TFTP and NFS](../g2l/002nfs.md)):

```bash
sudo cp arch/arm64/boot/Image arch/arm64/boot/dts/$DTB /srv/tftp/
```

Modules: install into a staging directory, then copy into the rootfs. `sudo make` would lose the cross-compile environment, so stage as a normal user first:

```bash
rm -rf /tmp/mod && make modules_install INSTALL_MOD_PATH=/tmp/mod INSTALL_MOD_STRIP=1
sudo cp -r /tmp/mod/lib/modules/* /nfs/rzg2l/lib/modules/
```

Check on the board that the running kernel and the modules match:

```bash
uname -r
ls /lib/modules/
```

## Out-of-tree build directory

Keep the source clean and build several configs from one tree with `O=`:

```bash
make O=~/work/build-g2l defconfig
make O=~/work/build-g2l -j$JOBS Image dtbs modules
```

All output paths above are then relative to `~/work/build-g2l`.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `aarch64-poky-linux-ld: unrecognized option '-Wl,-O1'` | Run `unset LDFLAGS` after sourcing the SDK |
| `make: aarch64-poky-linux-gcc: No such file or directory` | SDK environment not sourced in this shell |
| Board boots but `modprobe` fails with `Exec format error` / `invalid module format` | Modules built from a different config or version than the running `Image`. Install both from the same build |
| `uname -r` ends with `+` or `-dirty` | Tree has uncommitted changes. Harmless, but `/lib/modules/<release>` must match it exactly |
| Kernel hangs right after `Starting kernel ...` | Wrong dtb for the board, or `console=` missing in `bootargs`. Check the dtb name first |

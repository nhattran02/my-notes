# Custom layer & bbappend

All my changes (kernel patches, config, device tree, my own apps) go into my own layer instead of editing `meta-renesas`. The BSP layers stay untouched and can be upgraded, and my changes are kept in one place under version control.

## Variables

```bash
export BSP_DIR=~/yocto/rzg2l_vlp_v3.0.x
export LAYER=meta-mylayer
```

Run everything from the build directory:

```bash
cd $BSP_DIR && source poky/oe-init-build-env build
```

## Create and add the layer

```bash
bitbake-layers create-layer ../$LAYER
bitbake-layers add-layer ../$LAYER
bitbake-layers show-layers | grep mylayer
```

Generated structure:

```
meta-mylayer/
├── conf/layer.conf
├── COPYING.MIT
├── README
└── recipes-example/example/example_0.1.bb
```

Remove the example recipe:

```bash
rm -rf ../$LAYER/recipes-example
```

Check `LAYERSERIES_COMPAT` in `conf/layer.conf` matches the BSP's Yocto release:

=== "Dunfell"

    ```
    LAYERSERIES_COMPAT_meta-mylayer = "dunfell"
    ```

=== "Kirkstone"

    ```
    LAYERSERIES_COMPAT_meta-mylayer = "kirkstone"
    ```

=== "Scarthgap"

    ```
    LAYERSERIES_COMPAT_meta-mylayer = "scarthgap"
    ```

## Kernel: patches and config fragments

Layout:

```
meta-mylayer/recipes-kernel/linux/
├── linux-renesas_%.bbappend
└── files/
    ├── 0001-arm64-dts-renesas-enable-sci1.patch
    └── my-config.cfg
```

```bash
mkdir -p ../$LAYER/recipes-kernel/linux/files
```

`linux-renesas_%.bbappend` (the `%` matches any recipe version):

=== "Dunfell"

    ```
    FILESEXTRAPATHS_prepend := "${THISDIR}/files:"

    SRC_URI_append = " \
        file://0001-arm64-dts-renesas-enable-sci1.patch \
        file://my-config.cfg \
    "
    ```

=== "Kirkstone and later"

    ```
    FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

    SRC_URI:append = " \
        file://0001-arm64-dts-renesas-enable-sci1.patch \
        file://my-config.cfg \
    "
    ```

`my-config.cfg` holds only the options to change, in `.config` syntax:

```
CONFIG_SPI_SPIDEV=y
CONFIG_CAN_RCAR_CANFD=y
# CONFIG_DEBUG_INFO is not set
```

Rebuild and verify:

```bash
bitbake linux-renesas -c cleansstate && bitbake linux-renesas
grep -E "SPIDEV|CANFD" tmp/work/*/linux-renesas/*/build/.config
```

!!! tip
    Patches are applied with `git am`/`patch` in the order they appear in `SRC_URI`. Generate them with `git format-patch` from a kernel tree, or let `devtool finish` write them for you.

!!! warning
    If an option in a `.cfg` does not appear in the final `.config`, its dependencies are not met. BitBake prints a warning during `do_kernel_configcheck`. Find the missing dependency with `menuconfig` (search with `/`).

## U-Boot: patches

Same pattern, recipe name `u-boot`:

```bash
mkdir -p ../$LAYER/recipes-bsp/u-boot/files
```

`recipes-bsp/u-boot/u-boot_%.bbappend`:

=== "Dunfell"

    ```
    FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
    SRC_URI_append = " file://0001-change-default-bootcmd.patch"
    ```

=== "Kirkstone and later"

    ```
    FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
    SRC_URI:append = " file://0001-change-default-bootcmd.patch"
    ```

## My own application recipe

Layout:

```
meta-mylayer/recipes-apps/hello/
├── hello_1.0.bb
└── files/hello.c
```

```bash
mkdir -p ../$LAYER/recipes-apps/hello/files
```

`files/hello.c`:

```c
#include <stdio.h>

int main(void)
{
    printf("Hello from RZ\n");
    return 0;
}
```

`hello_1.0.bb`:

```
SUMMARY = "Hello world test application"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://hello.c"
S = "${WORKDIR}"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} hello.c -o hello
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 hello ${D}${bindir}/
}
```

!!! note
    Always use `${CC} ${CFLAGS} ${LDFLAGS}`. Calling `gcc` directly builds for the host, and leaving out `${LDFLAGS}` causes a `QA Issue: No GNU_HASH in the ELF binary` error.

Build it, then add it to the image:

```bash
bitbake hello
```

=== "Dunfell"

    ```
    IMAGE_INSTALL_append = " hello"
    ```

=== "Kirkstone and later"

    ```
    IMAGE_INSTALL:append = " hello"
    ```

## Install a file into the rootfs

For config files or scripts that are not built, e.g. `/etc/profile.d/myenv.sh`:

`recipes-core/myfiles/myfiles_1.0.bb`:

```
SUMMARY = "My rootfs files"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://myenv.sh"
S = "${WORKDIR}"

do_install() {
    install -d ${D}${sysconfdir}/profile.d
    install -m 0644 myenv.sh ${D}${sysconfdir}/profile.d/
}
```

## My own image recipe

Instead of growing `local.conf`, define an image in the layer.

`recipes-core/images/my-image.bb`:

```
require recipes-core/images/core-image-minimal.bb

SUMMARY = "Development image for RZ boards"

IMAGE_INSTALL_append = " \
    hello \
    i2c-tools \
    can-utils \
    ethtool \
    openssh \
"

IMAGE_FEATURES += "ssh-server-openssh debug-tweaks tools-debug"
```

(For Kirkstone and later, replace `IMAGE_INSTALL_append` with `IMAGE_INSTALL:append`.)

```bash
bitbake my-image
```

!!! warning
    `debug-tweaks` allows root login without a password. Use it only for development images.

## Keep the layer in git

```bash
cd ../$LAYER
git init
git add .
git commit -m "Initial layer"
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `No recipes available for: .../linux-renesas_%.bbappend` | File name does not match the recipe name. Check with `bitbake-layers show-recipes linux-renesas` |
| `Unable to get checksum for ... file://xxx` | File not found. Check `FILESEXTRAPATHS` and that the file is in `files/` |
| Patch fails: `Hunk #1 FAILED` | Patch made against a different kernel version. Regenerate it from the recipe's source (`devtool modify`) |
| `Layer ... is not compatible with the core layer` | Fix `LAYERSERIES_COMPAT` in `layer.conf` |
| `QA Issue: ... already-stripped` | Recipe strips the binary itself. Remove `-s` from the build or add `INSANE_SKIP_${PN} = "already-stripped"` (Kirkstone and later: `INSANE_SKIP:${PN}`) |

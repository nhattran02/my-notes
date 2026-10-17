# Build the BSP

Building a Renesas RZ BSP release from scratch, from the release packages to a bootable image.

## Variables

Set these first. Every command below uses them.

```bash
export BSP_DIR=~/yocto/rzg2l_vlp_v3.0.x      # where the BSP is extracted
export MACHINE=smarc-rzg2l                   # see the machine table below
export IMAGE=core-image-minimal              # or core-image-weston
```

| Board | `MACHINE` |
| ----- | --------- |
| RZ/G2L SMARC EVK | `smarc-rzg2l` |
| RZ/G2LC SMARC EVK | `smarc-rzg2lc` |
| RZ/G2UL SMARC EVK | `smarc-rzg2ul` |
| RZ/V2L SMARC EVK | `smarc-rzv2l` |
| RZ/G3S SMARC EVK | `smarc-rzg3s` |
| RZ/G3E SMARC EVK | `smarc-rzg3e` |
| RZ/V2N EVK | `rzv2n-evk` |

## Release packages

A BSP release from renesas.com is a set of zip files:

| Package | Content |
| ------- | ------- |
| BSP (VLP) | `poky`, `meta-openembedded`, `meta-renesas`, other base layers, and an offline `oss_pkg` download archive |
| Graphics library | `meta-rz-graphics` (GPU driver, needed for `core-image-weston`) |
| Video codec library | `meta-rz-codecs` (hardware video encode/decode) |

Graphics and codec packages are optional for `core-image-minimal`, and required for `core-image-weston` with GPU acceleration.

## Extract

```bash
mkdir -p $BSP_DIR && cd $BSP_DIR
unzip <path>/RTK0EF0045Z0021AZJ-v3.0.x.zip
tar -xf rzg_vlp_v3.0.x.tar.gz
```

Optional graphics and codec layers (they extract into `meta-rz-features/`):

```bash
cd $BSP_DIR
unzip <path>/<graphics-package>.zip
tar -xf meta-rz-features_graphics_v*.tar.gz
unzip <path>/<codec-package>.zip
tar -xf meta-rz-features_codec_v*.tar.gz
```

!!! note
    The exact archive names change between releases. Check `ls` after each `unzip` and follow the release note's file list.

Check the layers:

```bash
ls $BSP_DIR
```

```
meta-gplv2  meta-openembedded  meta-qt5  meta-renesas  meta-rz-features  meta-virtualization  poky
```

## Create the build directory

`TEMPLATECONF` points to the Renesas template for `local.conf` and `bblayers.conf`:

=== "RZ/G2L family (VLP 3.0.x, Dunfell)"

    ```bash
    cd $BSP_DIR
    TEMPLATECONF=$PWD/meta-renesas/meta-rz-common/docs/template/conf/ \
        source poky/oe-init-build-env build
    ```

=== "Older VLP (1.x)"

    ```bash
    cd $BSP_DIR
    TEMPLATECONF=$PWD/meta-renesas/meta-rzg2l/docs/template/conf/ \
        source poky/oe-init-build-env build
    ```

The shell is now in `$BSP_DIR/build`. In every new terminal, re-enter the build environment with (no `TEMPLATECONF` needed any more):

```bash
cd $BSP_DIR && source poky/oe-init-build-env build
```

!!! note "TODO"
    Newer BSPs for RZ/G3E and RZ/V2N use a different `TEMPLATECONF` path. Copy the exact line from that BSP's release note into a new tab here.

## Add optional layers

```bash
bitbake-layers add-layer ../meta-rz-features/meta-rz-graphics
bitbake-layers add-layer ../meta-rz-features/meta-rz-codecs
bitbake-layers show-layers
```

## Edit local.conf

Append to `conf/local.conf`:

```bash
cat >> conf/local.conf << EOF

# --- my settings ---
MACHINE = "${MACHINE}"
DL_DIR = "\${HOME}/yocto/downloads"
SSTATE_DIR = "\${HOME}/yocto/sstate-cache"
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"
# remove work files after each recipe is built (saves a lot of disk)
INHERIT += "rm_work"
RM_WORK_EXCLUDE += "linux-renesas u-boot"
EOF
```

!!! tip
    `RM_WORK_EXCLUDE` keeps the kernel and U-Boot work directories, so their source and build tree stay available for inspection and `devshell`.

### Offline build

The BSP ships the source tarballs of all packages (`oss_pkg`). To build without network access, extract them into `DL_DIR` and disable fetching:

```bash
tar -xf $BSP_DIR/oss_pkg_rzg_v3.0.x.7z -C ~/yocto/downloads    # or 7z x for .7z
echo 'BB_NO_NETWORK = "1"' >> conf/local.conf
```

## Build

```bash
bitbake $IMAGE
```

A first build takes 1–4 hours depending on the machine. Later builds reuse the sstate cache and take minutes.

Output files are in `tmp/deploy/images/$MACHINE/`.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `Nothing PROVIDES 'xxx'` | A layer is missing. Check `bitbake-layers show-layers`, add the layer providing the recipe |
| `Fetcher failure` / `Unable to fetch URL` | No network or server down. Use the [offline build](#offline-build) archive |
| `ERROR: Layer xxx is not compatible with the core layer` | Layer from a different Yocto release. Use layers from the same BSP release |
| `core-image-weston` builds but Weston fails to start (no GPU) | Graphics layer not added. Add `meta-rz-graphics` and rebuild |
| Build directory got messed up after changing many settings | Delete `tmp/` (not `sstate-cache`) and rebuild; sstate makes it fast |

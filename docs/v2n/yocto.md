# Yocto BSP build

## Fetch the layers

```bash
mkdir rzv2n-bsp && cd rzv2n-bsp
git clone https://git.yoctoproject.org/git/poky -b <yocto-branch>
git clone https://git.openembedded.org/meta-openembedded -b <yocto-branch>
git clone https://github.com/renesas-rz/meta-renesas -b <rzv-release>
# AI / DRP-AI support
git clone https://github.com/renesas-rz/meta-rz-features -b <rzv-release>
```

## Configure

```bash
source poky/oe-init-build-env
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-multimedia
bitbake-layers add-layer ../meta-renesas
bitbake-layers add-layer ../meta-rz-features/meta-rz-drpai
```

In `conf/local.conf`:

```text
MACHINE = "rzv2n-evk"
# enable DRP-AI + ISP support
MACHINE_FEATURES:append = " drpai isp"
# ACCEPT_EULA_rzv2n-evk = "1"
```

## Build

```bash
bitbake core-image-weston
bitbake core-image-weston -c populate_sdk     # cross SDK
```

## Output

`build/tmp/deploy/images/rzv2n-evk/`

| File                              | Purpose               |
| --------------------------------- | --------------------- |
| `Image`                           | kernel                |
| `r9a09g056n48-rzv2n-evk.dtb`      | device tree           |
| `bl2_bp_*.srec` / `fip_*.srec`    | bootloaders (SCIF)    |
| `*.wic.xz`                        | full SD-card image    |

## Tips

- Point `DL_DIR` / `SSTATE_DIR` at shared storage to speed up rebuilds.
- The DRP-AI userland (`libtvm_runtime`, translator headers) is pulled in by the
  `drpai` machine feature — verify with `bitbake -e core-image-weston | grep DRPAI`.

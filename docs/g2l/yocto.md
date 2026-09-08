# Yocto BSP build

## Fetch the layers

```bash
mkdir rzg2l-bsp && cd rzg2l-bsp
git clone https://git.yoctoproject.org/git/poky -b <yocto-branch>
git clone https://git.openembedded.org/meta-openembedded -b <yocto-branch>
git clone https://github.com/renesas-rz/meta-renesas -b <rzg-release>
```

## Configure

```bash
source poky/oe-init-build-env
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-multimedia
bitbake-layers add-layer ../meta-renesas
```

In `conf/local.conf`:

```text
MACHINE = "smarc-rzg2l"
# Accept the Renesas proprietary graphics/codec EULA if needed
# ACCEPT_EULA_smarc-rzg2l = "1"
```

## Build

```bash
bitbake core-image-weston
# minimal image:
bitbake core-image-minimal
# SDK / cross toolchain:
bitbake core-image-weston -c populate_sdk
```

## Output

`build/tmp/deploy/images/smarc-rzg2l/`

| File                          | Purpose                     |
| ----------------------------- | --------------------------- |
| `Image`                       | kernel                      |
| `r9a07g044l2-smarc.dtb`       | device tree                 |
| `bl2_bp-smarc-rzg2l_pmic.srec`| TF-A bl2 (SCIF flashing)    |
| `fip-smarc-rzg2l_pmic.srec`   | fip (bl31 + U-Boot)         |
| `*.wic.xz`                    | full SD-card image          |

## Tips

- Persistent shared cache: set `DL_DIR` and `SSTATE_DIR` outside the build tree.
- Rebuild one recipe: `bitbake -c cleansstate linux-renesas && bitbake linux-renesas`.

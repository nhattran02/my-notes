# Kernel & Device Tree

## Source

Renesas maintains the BSP kernel in
[`renesas-rz/rz_linux-cip`](https://github.com/renesas-rz/rz_linux-cip).

```bash
git clone https://github.com/renesas-rz/rz_linux-cip -b <branch>
export ARCH=arm64
export CROSS_COMPILE=aarch64-poky-linux-
make renesas_defconfig
make -j$(nproc) Image dtbs modules
```

## Device tree

| File                          | Role                                  |
| ----------------------------- | ------------------------------------- |
| `r9a07g044l2-smarc.dts`       | top-level board DT                    |
| `rzg2l-smarc-som.dtsi`        | SoM-level nodes (PMIC, eMMC, DDR)     |
| `rzg2l-smarc.dtsi`            | carrier-level nodes                   |
| `r9a07g044.dtsi`              | SoC description                       |

### Enabling a pin function

Most carrier headers are muxed. Add a `pinctrl` group and reference it from the
peripheral node, e.g. SCI:

```dts
&pinctrl {
    sci1_pins: sci1 {
        pinmux = <RZG2L_PORT_PINMUX(38, 0, 1)>, /* TX */
                 <RZG2L_PORT_PINMUX(38, 1, 1)>; /* RX */
    };
};

&sci1 {
    pinctrl-0 = <&sci1_pins>;
    pinctrl-names = "default";
    status = "okay";
};
```

## Out-of-tree modules

```bash
make -C <kernel> M=$PWD modules
# on target
insmod my_driver.ko
```

## Useful runtime checks

```bash
zcat /proc/config.gz | grep CONFIG_...
cat /sys/kernel/debug/pinctrl/*/pinmux-pins
cat /sys/kernel/debug/clk/clk_summary
```

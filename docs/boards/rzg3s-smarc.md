# RZ/G3S SMARC EVK

Renesas RZ/G3S SMARC Evaluation Board Kit (RZ SMARC Carrier Board II).

| Item              | Value                         |
| ----------------- | ----------------------------- |
| Device tree       | `r9a08g045s33-smarc.dtb`      |
| Boot mode switch  | SW_MODE (carrier board)       |
| Serial console    | 115200 bps                    |
| Kernel / DTB load | `0x48080000` / `0x48000000`   |

## Boot mode switches

SW_MODE[1..3] select the boot mode. SW_MODE[4] sets the minimum input current of the USB-C power adapter (ON in the Start-up Guide examples).

| SW_MODE[1] | SW_MODE[2] | SW_MODE[3] | Function                  |
| ---------- | ---------- | ---------- | ------------------------- |
| ON         | ON         | OFF        | eSD boot (3.3 V start-up) |
| OFF        | ON         | OFF        | SCIF download             |
| ON         | OFF        | OFF        | eMMC boot (1.8 V)         |
| OFF        | OFF        | OFF        | SPI boot (1.8 V)          |

Useful SW_CONFIG settings (on the module):

| SW_CONFIG | Signal          | ON                       | OFF                      |
| --------- | --------------- | ------------------------ | ------------------------ |
| 6         | RZ_BOOTCPUSEL   | Cold boot from Cortex-M33 | Cold boot from Cortex-A55 |
| 3         | SW_SD2_EN       | SD2 disabled             | SD2 enabled              |
| 2         | SW_SD0_DEV_SEL  | SD0 connected to uSD0    | SD0 connected to eMMC    |
| 1         | RZ_DEBUGEN      | JTAG debug disabled      | JTAG debug enabled       |

## Bootloader flash addresses

| File                          | Program Top | Save Address |
| ----------------------------- | ----------- | ------------ |
| `bl2_bp_spi-smarc-rzg3s.srec` | `A1E00`     | `00000`      |
| `fip-smarc-rzg3s.srec`        | `00000`     | `60000`      |

Flash Writer: `FlashWriter-smarc-rzg3s.mot`. See [Flashing the bootloader](../guides/flash-bootloader.md).

## MMC devices

U-Boot: `sd@11c00000: 0, sd@11c10000: 1, sd@11c20000: 2`. SD0 goes to the eMMC or to uSD0 depending on SW_CONFIG[2].

## Kernel sources

Device tree:

```
arch/arm64/boot/dts/renesas/r9a08g045s33-smarc.dts
arch/arm64/boot/dts/renesas/rzg3s-smarc-som.dtsi
arch/arm64/boot/dts/renesas/rzg3s-smarc.dtsi
arch/arm64/boot/dts/renesas/r9a08g045s33.dtsi
arch/arm64/boot/dts/renesas/r9a08g045.dtsi
arch/arm64/boot/dts/renesas/rzg3s-smarc-switches.h
arch/arm64/boot/dts/renesas/rz-smarc-pmod1-type-3a.dtsi
```

Bindings:

```
Documentation/devicetree/bindings/serial/renesas,scif.yaml
Documentation/devicetree/bindings/serial/serial.yaml
```

## Module notes

- [MTU3 PWM](../bsp/mtu3-pwm.md): channel map of the `pwm-rz-mtu3` driver.

## Sources

- [SMARC EVK of RZ/G3S Linux Start-up Guide](https://www.renesas.com/document/gde/smarc-evk-rzg3s-linux-start-guide) (R01US0816EJ0101, Rev.1.01)

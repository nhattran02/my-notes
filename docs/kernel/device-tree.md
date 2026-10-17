# Device tree

Reading, modifying and debugging the device tree of RZ boards.

## Variables

```bash
export KSRC=~/work/rz_linux-cip
export DTS_DIR=$KSRC/arch/arm64/boot/dts/renesas
export BOARD_DTS=r9a07g044l2-smarc        # without extension
```

## File structure

A board `.dts` is built from several include files: SoC → SoM → carrier board. List what a board includes:

```bash
grep -n "#include" $DTS_DIR/$BOARD_DTS.dts
```

Typical layering for RZ SMARC boards:

| Level | RZ/G2L example | Content |
| ----- | -------------- | ------- |
| SoC family | `r9a07g044.dtsi` | All on-chip peripherals, disabled by default |
| SoC variant | `r9a07g044l2.dtsi` | CPU count / package-specific parts |
| SoM | `rzg2l-smarc-som.dtsi` | Parts on the module: DDR, PMIC, eMMC, Ethernet PHY |
| Carrier | `rzg2l-smarc.dtsi` | Connectors on the carrier board: SD slot, audio, CAN, PMOD |
| Board | `r9a07g044l2-smarc.dts` | Top level, ties everything together |

The RZ/G3S file list is in the [RZ/G3S page](../g3s/index.md).

!!! tip
    Do not search through all the includes by hand. Build the dtb and decompile it to see the final merged tree (next section).

## See the final tree

Build and decompile:

```bash
make renesas/$BOARD_DTS.dtb
dtc -I dtb -O dts -o /tmp/$BOARD_DTS.dts $DTS_DIR/$BOARD_DTS.dtb 2>/dev/null
less /tmp/$BOARD_DTS.dts
```

The preprocessed source (all `#include` and macros expanded, labels kept) is also left in the build tree:

```bash
less $DTS_DIR/.$BOARD_DTS.dtb.dts.tmp
```

## Common modifications

Put board changes in the board `.dts` (or my own `.dts` that includes it), and refer to existing nodes by label with `&label`. Do not edit the SoC `.dtsi`.

### Enable a peripheral

```dts
&spi1 {
    pinctrl-0 = <&spi1_pins>;
    pinctrl-names = "default";
    status = "okay";
};
```

### Disable a peripheral

```dts
&canfd {
    status = "disabled";
};
```

### Pin multiplexing (RZ/G2L family)

Pin functions are set with `RZG2L_PORT_PINMUX(port, pin, function)` from `include/dt-bindings/pinctrl/rzg2l-pinctrl.h`. Port, pin and function number come from the pin function table in the SoC hardware manual.

```dts
#include <dt-bindings/pinctrl/rzg2l-pinctrl.h>

&pinctrl {
    spi1_pins: spi1 {
        pinmux = <RZG2L_PORT_PINMUX(<port>, <pin>, <func>)>,   /* CK */
                 <RZG2L_PORT_PINMUX(<port>, <pin>, <func>)>,   /* MOSI */
                 <RZG2L_PORT_PINMUX(<port>, <pin>, <func>)>,   /* MISO */
                 <RZG2L_PORT_PINMUX(<port>, <pin>, <func>)>;   /* SSL */
    };
};
```

!!! warning
    Many carrier-board pins are shared between functions (selected by DIP switches on the SMARC carrier). Check the board schematic and the switch settings before assigning a pin.

### GPIO: LED and button

```dts
#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/input/input.h>

/ {
    leds {
        compatible = "gpio-leds";
        led-user {
            gpios = <&pinctrl RZG2L_GPIO(<port>, <pin>) GPIO_ACTIVE_HIGH>;
            linux,default-trigger = "heartbeat";
        };
    };

    keys {
        compatible = "gpio-keys";
        key-user {
            gpios = <&pinctrl RZG2L_GPIO(<port>, <pin>) GPIO_ACTIVE_LOW>;
            linux,code = <KEY_ENTER>;
            label = "user-key";
        };
    };
};
```

### I2C device

```dts
&i2c1 {
    status = "okay";

    eeprom@50 {
        compatible = "atmel,24c02";
        reg = <0x50>;
    };
};
```

### Remove nodes or properties

```dts
&i2c1 {
    /delete-node/ codec@1a;
};

&sdhi1 {
    /delete-property/ cd-gpios;
};
```

## Inspect the live tree on the board

The running tree is exposed under `/proc/device-tree` (a link to `/sys/firmware/devicetree/base`):

```bash
cat /proc/device-tree/model; echo
ls /proc/device-tree/soc/
```

Check a node's status and properties:

```bash
cat /proc/device-tree/soc/spi@1004b000/status; echo
hexdump -C /proc/device-tree/soc/spi@1004b000/reg
```

Dump the whole live tree as source (if `dtc` is installed on the board):

```bash
dtc -I fs -O dts /proc/device-tree 2>/dev/null | less
```

## Is the driver bound?

```bash
ls /sys/bus/platform/devices/ | grep spi
ls -l /sys/bus/platform/devices/1004b000.spi/driver
```

A missing `driver` link means no driver was bound. Check:

```bash
dmesg | grep -i 1004b000
cat /sys/kernel/debug/devices_deferred      # probe waiting for a resource (clock, GPIO, regulator)
```

Pin mux state (requires `CONFIG_DEBUG_FS`):

```bash
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -v UNCLAIMED
```

## Validate against bindings

Bindings (the rules each node must follow) are in `Documentation/devicetree/bindings/`. Find the binding for a `compatible`:

```bash
grep -rl "renesas,rspi" $KSRC/Documentation/devicetree/bindings/
```

Check a dtb against the schemas:

```bash
pip3 install --user dtschema
make dtbs_check DT_SCHEMA_FILES=spi/renesas,rspi.yaml
```

## Device tree overlays in U-Boot

Apply a small `.dtbo` on top of the base dtb at boot, without rebuilding the base dtb. The base dtb must be built with symbols (`DTC_FLAGS=-@`).

Overlay source `my-overlay.dtso`:

```dts
/dts-v1/;
/plugin/;

&spi1 {
    status = "okay";
};
```

Build:

```bash
make DTC_FLAGS=-@ renesas/$BOARD_DTS.dtb
dtc -@ -I dts -O dtb -o my-overlay.dtbo my-overlay.dtso
```

In U-Boot (example with TFTP):

```
tftp 0x48000000 r9a07g044l2-smarc.dtb
fdt addr 0x48000000
fdt resize 8192
tftp 0x48010000 my-overlay.dtbo
fdt apply 0x48010000
tftp 0x48080000 Image
booti 0x48080000 - 0x48000000
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `Error: ... syntax error` with a line number in `.dtb.dts.tmp` | Look up that line in the preprocessed file, not the original `.dts` |
| `Reference to non-existent node or label "xxx"` | Label defined in a file not included, or misspelled. Search with `grep -rn "xxx:" $DTS_DIR` |
| Node is `okay` but device missing in Linux | Driver not enabled in the kernel config, or probe deferred. Check `devices_deferred` and `dmesg` |
| `pin ... already requested by ...` in dmesg | Two nodes claim the same pin. Disable the other user or change the pinmux |
| `fdt apply` fails with `FDT_ERR_NOTFOUND` | Base dtb built without `-@`, so labels are not available to the overlay |

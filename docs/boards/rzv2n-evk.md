# RZ/V2N EVK

Renesas RZ/V2N Evaluation Board Kit.

| Item              | Value                         |
| ----------------- | ----------------------------- |
| Device tree       | `r9a09g056n48-rzv2n-evk.dtb`  |
| Boot mode switch  | DSW1                          |
| Serial console    | 115200 bps, 8N1               |
| Power             | SW3 ON, then SW2 ON to power on |

## Boot modes (DSW1)

| Boot mode | Function        |
| --------- | --------------- |
| 1         | eMMC boot       |
| 2         | xSPI boot (normal) |
| 3         | SCIF download   |

The switch positions for each mode are shown as figures in the RZ/V AI SDK guide.

## Bootloader flash addresses

| File                         | Program Top | Save Address |
| ---------------------------- | ----------- | ------------ |
| `bl2_bp_spi-rzv2n-evk.srec`  | `8101E00`   | `00000`      |
| `fip-rzv2n-evk.srec`         | `00000`     | `60000`      |

Flash Writer: `Flash_Writer_SCIF_RZV2N_DEV_LPDDR4X.mot`. See [Flashing the bootloader](../guides/flash-bootloader.md).

## MMC devices

| Medium | Linux device   |
| ------ | -------------- |
| eMMC   | `/dev/mmcblk0` |

## Sources

- [RZ/V AI SDK dev guide, D3 "How to boot from xSPI" and D4 "How to boot from eMMC"](https://renesas-rz.github.io/rzv_ai_sdk/8.10/dev_guide.html) (Rev. 8.10)

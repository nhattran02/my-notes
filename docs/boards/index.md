# Boards

Quick reference for each board: device tree, boot switches, flash addresses and anything else that is specific to the board. Procedures that work on every board are in [Guides](../guides/index.md).

| Board            | SoC         | Device tree                  | Boot switch | Page                        |
| ---------------- | ----------- | ---------------------------- | ----------- | --------------------------- |
| RZ/G2L SMARC EVK | R9A07G044L2 | `r9a07g044l2-smarc.dtb`      | SW11        | [RZ/G2L SMARC](rzg2l-smarc.md) |
| RZ/G3S SMARC EVK | R9A08G045S33| `r9a08g045s33-smarc.dtb`     | SW_MODE     | [RZ/G3S SMARC](rzg3s-smarc.md) |
| RZ/V2N EVK       | R9A09G056N48| `r9a09g056n48-rzv2n-evk.dtb` | DSW1        | [RZ/V2N EVK](rzv2n-evk.md)     |

Other boards seen so far:

| Board            | Device tree              |
| ---------------- | ------------------------ |
| RZ/G3E SMARC EVK | `r9a09g047e57-smarc.dtb` |

## Adding a board

1. Copy one of the board pages and fill in what you know. Leave unknown values out rather than guessing.
2. Add a row to the table above.
3. In the [Guides](../guides/index.md), add a tab with the board name (for example `=== "RZ/V2H"`) wherever the steps differ.

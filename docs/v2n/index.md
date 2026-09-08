# RZ/V2N EVK

Bring-up and development notes for the Renesas RZ/V2N Evaluation Board Kit.

## Board at a glance

| Item        | Value                                                       |
| ----------- | ---------------------------------------------------------- |
| SoC         | RZ/V2N (R9A09G056) — 4× Cortex-A55, 1× Cortex-M33          |
| AI          | DRP-AI3 (~15 TOPS), hardware ISP                           |
| RAM         | LPDDR4X                                                    |
| Storage     | eMMC, microSD, QSPI NOR                                    |
| I/O         | GbE, USB 3.2, MIPI CSI-2, MIPI DSI / HDMI                  |
| Console     | USB-serial, 115200 8N1                                    |

## Sections

- [Board setup](setup.md) — cabling, serial console, SD imaging
- [Boot modes & flashing](boot.md) — DIP switches, SCIF download, Flash Writer
- [Yocto BSP build](yocto.md) — layers, `MACHINE=rzv2n-evk`, DRP-AI/ISP options
- [DRP-AI development](drpai.md) — DRP-AI TVM toolchain, model conversion, runtime
- [Peripherals](peripherals.md) — CSI-2 camera + ISP, display, networking, GPIO

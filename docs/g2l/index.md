# RZ/G2L SMARC

Bring-up and development notes for the Renesas RZ/G2L SMARC Evaluation Board Kit.

## Board at a glance

| Item        | Value                                                        |
| ----------- | ----------------------------------------------------------- |
| SoC         | RZ/G2L (R9A07G044L) — 2× Cortex-A55 @ 1.2 GHz, 1× Cortex-M33 |
| GPU / VPU   | Mali-G31, H.264 encode/decode                               |
| RAM         | 2 GB DDR4                                                   |
| Storage     | eMMC, microSD, QSPI NOR                                     |
| Form factor | SMARC 2.1 SoM + carrier board                              |
| Console     | USB-serial, 115200 8N1                                     |

## Sections

- [Board setup](setup.md) — cabling, serial console, power
- [Boot modes & flashing](boot.md) — DIP switches, SCIF download, Flash Writer
- [Yocto BSP build](yocto.md) — layers, `MACHINE=smarc-rzg2l`, images
- [Kernel & Device Tree](kernel.md) — defconfig, DT overlays, out-of-tree modules
- [Peripherals](peripherals.md) — CSI-2 camera, DSI/HDMI, CAN, I2C, SPI, GPIO
- [Cortex-M33 / remoteproc](cm33.md) — building and loading CM33 firmware

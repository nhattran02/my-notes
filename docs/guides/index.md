# Guides

Step-by-step procedures. Each guide works on every board; where a step differs, the page has a tab per board. Choosing a board in one tab switches the tabs on every page.

| Guide | When to use it |
| ----- | -------------- |
| [Flashing the bootloader](flash-bootloader.md) | New board, or after rebuilding TF-A / U-Boot |
| [Booting with TFTP and NFS](nfs-boot.md) | Daily development: kernel and rootfs stay on the host |
| [Booting from SD card and eMMC](sd-emmc-boot.md) | Standalone boot without the host |

Typical order on a new board: flash the bootloader, boot over TFTP/NFS, then (optionally) write the images to SD or eMMC.

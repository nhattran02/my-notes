# Cheatsheet

The commands I copy most often, on one page. Each section links to the full article.

## Boards

| Board | Yocto `MACHINE` | Device tree |
| ----- | --------------- | ----------- |
| RZ/G2L SMARC EVK | `smarc-rzg2l` | `r9a07g044l2-smarc.dtb` |
| RZ/G3S SMARC EVK | `smarc-rzg3s` | `r9a08g045s33-smarc.dtb` |
| RZ/G3E SMARC EVK | `smarc-rzg3e` | `r9a09g047e57-smarc.dtb` |
| RZ/V2N EVK | `rzv2n-evk` | `r9a09g056n48-rzv2n-evk.dtb` |

RZ/G2L: kernel at `0x48080000`, dtb at `0x48000000`, console `ttySC0` at 115200 8N1. eMMC = `mmc 0` / `mmcblk0`, micro SD = `mmc 1` / `mmcblk1`.

## Network setup used in these notes

| | IP |
| - | -- |
| Host (TFTP / NFS server) | `192.168.1.3` |
| Board | `192.168.1.5` |
| TFTP directory | `/srv/tftp` |
| NFS rootfs | `/nfs/rzg2l` |

## U-Boot

Full details: [Environment presets](uboot/env-presets.md), [U-Boot commands](uboot/commands.md).

```
setenv ipaddr 192.168.1.5
setenv serverip 192.168.1.3
setenv netmask 255.255.255.0
setenv nfsdir /nfs/rzg2l
setenv kernel Image
setenv fdtfile r9a07g044l2-smarc.dtb
setenv k_addr 0x48080000
setenv fdt_addr 0x48000000
setenv args_common 'console=ttySC0,115200n8 rootwait rw noinitrd nohlt panic=1 earlyprintk=ttySC0,115200n8'
setenv nfsboot 'setenv bootargs ${args_common} ip=${ipaddr}:${serverip}::${netmask}::eth0 root=/dev/nfs nfsroot=${serverip}:${nfsdir},nfsvers=3,tcp; tftp ${k_addr} ${kernel}; tftp ${fdt_addr} ${fdtfile}; booti ${k_addr} - ${fdt_addr}'
setenv sdboot 'setenv bootargs ${args_common} root=/dev/mmcblk1p2; mmc dev 1; ext4load mmc 1:1 ${k_addr} ${kernel}; ext4load mmc 1:1 ${fdt_addr} ${fdtfile}; booti ${k_addr} - ${fdt_addr}'
setenv emmcboot 'setenv bootargs ${args_common} root=/dev/mmcblk0p2; mmc dev 0; ext4load mmc 0:1 ${k_addr} ${kernel}; ext4load mmc 0:1 ${fdt_addr} ${fdtfile}; booti ${k_addr} - ${fdt_addr}'
setenv bootcmd 'run nfsboot'
saveenv
```

```
run sdboot                         # boot another method once
printenv bootcmd bootargs
env default -a; saveenv            # reset environment
```

## Yocto

Full details: [Build the BSP](yocto/build-bsp.md), [BitBake commands](yocto/bitbake.md).

```bash
cd $BSP_DIR && source poky/oe-init-build-env build
bitbake core-image-minimal
bitbake core-image-minimal -c populate_sdk
```

```bash
bitbake linux-renesas -c cleansstate && bitbake linux-renesas
bitbake virtual/kernel -c menuconfig
bitbake virtual/kernel -c diffconfig
bitbake -e linux-renesas | grep -E "^(SRCREV|KBRANCH|WORKDIR)="
bitbake-layers show-appends | grep linux-renesas
oe-pkgdata-util find-path /usr/sbin/i2cdetect
```

Output directory:

```bash
ls $BSP_DIR/build/tmp/deploy/images/$MACHINE/
```

devtool ([devtool](yocto/devtool.md)):

```bash
devtool modify linux-renesas
# edit + git commit in build/workspace/sources/linux-renesas
bitbake linux-renesas
devtool finish linux-renesas ../meta-mylayer
```

## Host: deploy to TFTP / NFS

```bash
export DEPLOY=$BSP_DIR/build/tmp/deploy/images/$MACHINE
sudo cp -L $DEPLOY/Image $DEPLOY/r9a07g044l2-smarc.dtb /srv/tftp/
sudo rm -rf /nfs/rzg2l/* && sudo tar -xf $DEPLOY/core-image-minimal-$MACHINE.tar.bz2 -C /nfs/rzg2l
```

```bash
sudo systemctl restart tftpd-hpa nfs-kernel-server
showmount -e localhost
```

## Kernel outside Yocto

Full details: [Build the kernel](kernel/build.md), [Fast edit–test loop](kernel/fast-iteration.md).

```bash
source $SDK_DIR/environment-setup-aarch64-poky-linux && unset LDFLAGS
make defconfig
make -j$(nproc) Image dtbs modules
make renesas/r9a07g044l2-smarc.dtb
sudo cp arch/arm64/boot/Image arch/arm64/boot/dts/renesas/r9a07g044l2-smarc.dtb /srv/tftp/
```

```bash
scripts/config --enable CONFIG_SPI_SPIDEV && make olddefconfig
git format-patch $BASE -o ~/work/patches
./scripts/checkpatch.pl --strict -g HEAD
```

## SD card

Full details: [Flash full SD image](g2l/004wic.md), [Booting from SD Card and eMMC](g2l/003sd_emmc.md).

```bash
lsblk -d -o NAME,SIZE,MODEL,TRAN
sudo bmaptool copy $DEPLOY/core-image-minimal-$MACHINE.wic.gz /dev/sdX
```

## Board

Full details: [Network & file transfer](debug/network.md), [Peripherals from user space](debug/peripherals.md), [Kernel debugging](debug/kernel-debug.md).

```bash
ip link set eth0 up && ip addr add 192.168.1.5/24 dev eth0
ssh-keygen -R 192.168.1.5          # on the host, after reflashing
```

```bash
uname -a
cat /proc/device-tree/model; echo
zcat /proc/config.gz | grep CONFIG_XXX
dmesg -T --level=err,warn
cat /sys/kernel/debug/devices_deferred
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -v UNCLAIMED
echo 'file drivers/spi/spi-rspi.c +p' > /sys/kernel/debug/dynamic_debug/control
```

```bash
gpioinfo | head; gpioset gpiochip0 <line>=1        # RZ/G2L line = port * 8 + pin
i2cdetect -y 1
spidev_test -D /dev/spidev1.0 -s 1000000 -v
ip link set can0 type can bitrate 500000 && ip link set can0 up && candump can0
devmem2 <address> w
```

```bash
cat /sys/class/thermal/thermal_zone*/temp
free -h; grep Cma /proc/meminfo
systemd-analyze blame | head
```

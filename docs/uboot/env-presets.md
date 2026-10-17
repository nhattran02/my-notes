# Environment presets

Instead of retyping `bootcmd` and `bootargs` every time I switch boot method, I store each method as a named script in the U-Boot environment. Switching is then one `run` command.

The individual boot methods are explained in [Booting with TFTP and NFS](../g2l/002nfs.md) and [Booting from SD Card and eMMC](../g2l/003sd_emmc.md). This page only combines them.

## 1. Common settings

Board-specific values, set once. Values below are for RZ/G2L SMARC; change `fdtfile` for other boards.

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
```

!!! note
    `ethaddr` (MAC address) is usually already set from the board. If `printenv ethaddr` shows nothing, set it once with `setenv ethaddr xx:xx:xx:xx:xx:xx`. U-Boot allows setting it only once; changing it later needs `env default -a`.

## 2. Boot scripts

Copy these three lines as they are. The single quotes keep `${...}` unexpanded until the script runs, so later changes to the common settings take effect.

TFTP kernel + NFS rootfs:

```
setenv nfsboot 'setenv bootargs ${args_common} ip=${ipaddr}:${serverip}::${netmask}::eth0 root=/dev/nfs nfsroot=${serverip}:${nfsdir},nfsvers=3,tcp; tftp ${k_addr} ${kernel}; tftp ${fdt_addr} ${fdtfile}; booti ${k_addr} - ${fdt_addr}'
```

SD card (partition 1: kernel + dtb, partition 2: rootfs):

```
setenv sdboot 'setenv bootargs ${args_common} root=/dev/mmcblk1p2; mmc dev 1; ext4load mmc 1:1 ${k_addr} ${kernel}; ext4load mmc 1:1 ${fdt_addr} ${fdtfile}; booti ${k_addr} - ${fdt_addr}'
```

eMMC (same layout):

```
setenv emmcboot 'setenv bootargs ${args_common} root=/dev/mmcblk0p2; mmc dev 0; ext4load mmc 0:1 ${k_addr} ${kernel}; ext4load mmc 0:1 ${fdt_addr} ${fdtfile}; booti ${k_addr} - ${fdt_addr}'
```

Optional: TFTP kernel + rootfs on the SD card. Useful when only the kernel changes and the SD rootfs is fine:

```
setenv tftpsd 'setenv bootargs ${args_common} root=/dev/mmcblk1p2; tftp ${k_addr} ${kernel}; tftp ${fdt_addr} ${fdtfile}; booti ${k_addr} - ${fdt_addr}'
```

## 3. Choose the default and save

```
setenv bootcmd 'run nfsboot'
saveenv
```

## Switching

Permanently switch the default boot method:

```
setenv bootcmd 'run sdboot'
saveenv
```

Boot another method once, without changing the default (stop autoboot first):

```
run emmcboot
```

Check what is stored:

```
printenv bootcmd nfsboot sdboot emmcboot
```

## Reset to factory environment

```
env default -a
saveenv
```

This removes all presets above, and `ethaddr` if it is not stored elsewhere.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| Characters missing after pasting a long `setenv` line | The serial console drops characters on fast paste. In Tera Term set *Setup → Serial port → Transmit delay* to 1–5 ms/char, or paste line by line |
| `## Error: "nfsboot" not defined` | The script was not saved, or `env default -a` was run. Set it again and `saveenv` |
| Variables expanded too early (fixed IPs inside the script) | Script was set with double quotes. Use single quotes |
| `Can't overwrite "ethaddr"` | MAC can be set only once. `env default -a`, then set it again |
| Kernel panics `VFS: Unable to mount root fs` | Wrong `root=` for the chosen medium. Compare `printenv bootargs` after `run` with the partition layout |

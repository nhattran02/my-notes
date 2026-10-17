# Kernel config

Finding, changing and keeping kernel configuration options.

## Variables

```bash
export KSRC=~/work/rz_linux-cip
export SDK_DIR=/opt/poky/3.1.x
```

```bash
source $SDK_DIR/environment-setup-aarch64-poky-linux && unset LDFLAGS
cd $KSRC
```

## Interactive: menuconfig

```bash
sudo apt-get install -y libncurses-dev     # once
make menuconfig
```

Inside menuconfig:

| Key | Action |
| --- | ------ |
| `/` | Search for a symbol (without `CONFIG_`), shows location and dependencies |
| `1`, `2`, ... | In search results: jump to that entry |
| `y` / `m` / `n` | Built-in / module / disabled |
| `?` | Help for the selected option, including `Depends on:` and `Selected by:` |

## Scripted: scripts/config

Faster than menuconfig when I already know the symbol names, and easy to copy:

```bash
scripts/config --enable  CONFIG_SPI_SPIDEV
scripts/config --module  CONFIG_CAN_RCAR_CANFD
scripts/config --disable CONFIG_DEBUG_INFO
scripts/config --set-val CONFIG_LOG_BUF_SHIFT 18
scripts/config --set-str CONFIG_LOCALVERSION "-mytest"
make olddefconfig          # resolve dependencies after editing
```

Check the result:

```bash
grep -E "SPIDEV|CANFD|LOCALVERSION=" .config
```

!!! warning
    `scripts/config` writes the value even if dependencies are not met; `make olddefconfig` may then silently turn it off again. Always grep `.config` after `olddefconfig`.

## Which option builds a driver?

From a source file name to its config symbol:

```bash
grep -rn "sh-sci.o" drivers/tty/serial/Makefile
```

```
obj-$(CONFIG_SERIAL_SH_SCI) += sh-sci.o
```

From a device tree `compatible` string to the driver:

```bash
grep -rln '"renesas,scif-r9a07g044"' drivers/
```

## Save and compare configs

Minimal defconfig (only the non-default values):

```bash
make savedefconfig
cp defconfig arch/arm64/configs/my_defconfig
make my_defconfig          # later: load it again
```

Compare two configs:

```bash
scripts/diffconfig .config.old .config
```

```
 SPI_SPIDEV n -> y
+CAN_RCAR_CANFD m
```

## Config fragments

A fragment is a small file with only the options I change. Merge it on top of a base config:

```bash
cat > ~/work/my-debug.cfg << 'EOF'
CONFIG_IKCONFIG=y
CONFIG_IKCONFIG_PROC=y
CONFIG_DEBUG_FS=y
CONFIG_DYNAMIC_DEBUG=y
EOF

scripts/kconfig/merge_config.sh -m .config ~/work/my-debug.cfg
make olddefconfig
```

The same `.cfg` file goes into Yocto through a `.bbappend` (see [Custom layer & bbappend](../yocto/custom-layer.md)).

## Read the config on the board

Requires `CONFIG_IKCONFIG_PROC=y`:

```bash
zcat /proc/config.gz | grep -E "SPIDEV|CANFD"
```

Without it, check the module directory instead:

```bash
ls /lib/modules/$(uname -r)/kernel/drivers/spi/
```

## Options I usually enable for development

```
# read the running config from /proc/config.gz
CONFIG_IKCONFIG=y
CONFIG_IKCONFIG_PROC=y
# debugfs: pinctrl, clk, regmap, gpio info under /sys/kernel/debug
CONFIG_DEBUG_FS=y
# turn pr_debug()/dev_dbg() on at runtime
CONFIG_DYNAMIC_DEBUG=y
# user-space access to SPI / I2C
CONFIG_SPI_SPIDEV=y
CONFIG_I2C_CHARDEV=y
# /dev/mem for devmem2 (register peek/poke)
CONFIG_DEVMEM=y
# readable stack traces
CONFIG_KALLSYMS=y
CONFIG_KALLSYMS_ALL=y
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| Option disappears after `make olddefconfig` | A dependency is off. Search it in menuconfig (`/`) and enable what `Depends on:` lists |
| Option not shown in menuconfig | Its `depends on` is false, or wrong `ARCH`. Make sure `ARCH=arm64` is set |
| `/proc/config.gz` missing | `CONFIG_IKCONFIG_PROC` not set in the running kernel |
| Yocto ignores my changes made with `make menuconfig` in the work dir | They are lost on rebuild. Put them in a `.cfg` fragment in my layer |

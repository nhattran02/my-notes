# Kernel module

Templates for out-of-tree kernel modules: a minimal module, a platform driver bound from the device tree, and how to build them with a kernel tree, the SDK, or Yocto.

## Variables

```bash
export KDIR=~/work/rz_linux-cip       # a configured and built kernel tree
export SDK_DIR=/opt/poky/3.1.x
export BOARD_IP=192.168.1.5
```

!!! warning
    The module must be built against the **same kernel** (version and config) that runs on the board, otherwise loading fails with `invalid module format`.

## Minimal module

`hello.c`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/init.h>

static int count = 1;
module_param(count, int, 0644);
MODULE_PARM_DESC(count, "Number of greetings");

static int __init hello_init(void)
{
	int i;

	for (i = 0; i < count; i++)
		pr_info("hello: init %d\n", i);
	return 0;
}

static void __exit hello_exit(void)
{
	pr_info("hello: exit\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Nhat Tran");
MODULE_DESCRIPTION("Hello world module");
```

`Makefile` (works for all build methods below):

```make
obj-m := hello.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

modules_install:
	$(MAKE) -C $(KDIR) M=$(PWD) modules_install

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

## Build

=== "Against my kernel tree"

    ```bash
    source $SDK_DIR/environment-setup-aarch64-poky-linux && unset LDFLAGS
    make KDIR=$KDIR
    ```

=== "Against the SDK kernel source"

    Requires `kernel-devsrc` in the SDK (see [SDK](../yocto/sdk.md)). Prepare the kernel source once:

    ```bash
    source $SDK_DIR/environment-setup-aarch64-poky-linux && unset LDFLAGS
    cd $SDKTARGETSYSROOT/usr/src/kernel
    sudo -E make scripts prepare
    cd -
    ```

    Then build the module:

    ```bash
    make KDIR=$SDKTARGETSYSROOT/usr/src/kernel
    ```

Check the result:

```bash
modinfo hello.ko
```

```
vermagic:       5.10.xxx-cip-yocto-standard SMP preempt mod_unload aarch64
```

`vermagic` must match `uname -r` on the board.

## Load on the board

```bash
scp hello.ko root@$BOARD_IP:/home/root/
```

On the board:

```bash
insmod hello.ko count=3
dmesg | tail
cat /sys/module/hello/parameters/count
rmmod hello
```

Install it so `modprobe` can find it (and load dependencies automatically):

```bash
mkdir -p /lib/modules/$(uname -r)/extra
cp hello.ko /lib/modules/$(uname -r)/extra/
depmod -a
modprobe hello
```

Load at boot (systemd):

```bash
echo hello > /etc/modules-load.d/hello.conf
```

## Platform driver template

A driver probed from a device tree node. Covers the parts I need most often: matching, register mapping, clock, IRQ, and a sysfs attribute.

`mydrv.c`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/clk.h>
#include <linux/interrupt.h>
#include <linux/io.h>
#include <linux/module.h>
#include <linux/of.h>
#include <linux/platform_device.h>

struct mydrv {
	struct device *dev;
	void __iomem *base;
	struct clk *clk;
	int irq;
};

static irqreturn_t mydrv_irq(int irq, void *data)
{
	struct mydrv *priv = data;

	dev_dbg(priv->dev, "irq\n");
	return IRQ_HANDLED;
}

static ssize_t reg0_show(struct device *dev, struct device_attribute *attr,
			 char *buf)
{
	struct mydrv *priv = dev_get_drvdata(dev);

	return sprintf(buf, "0x%08x\n", readl(priv->base));
}
static DEVICE_ATTR_RO(reg0);

static int mydrv_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct mydrv *priv;
	int ret;

	priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;
	priv->dev = dev;

	priv->base = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(priv->base))
		return PTR_ERR(priv->base);

	priv->clk = devm_clk_get(dev, NULL);
	if (IS_ERR(priv->clk))
		return dev_err_probe(dev, PTR_ERR(priv->clk), "no clock\n");
	ret = clk_prepare_enable(priv->clk);
	if (ret)
		return ret;

	priv->irq = platform_get_irq(pdev, 0);
	if (priv->irq < 0) {
		ret = priv->irq;
		goto err_clk;
	}
	ret = devm_request_irq(dev, priv->irq, mydrv_irq, 0, dev_name(dev), priv);
	if (ret)
		goto err_clk;

	platform_set_drvdata(pdev, priv);

	ret = device_create_file(dev, &dev_attr_reg0);
	if (ret)
		goto err_clk;

	dev_info(dev, "probed, irq %d\n", priv->irq);
	return 0;

err_clk:
	clk_disable_unprepare(priv->clk);
	return ret;
}

static int mydrv_remove(struct platform_device *pdev)
{
	struct mydrv *priv = platform_get_drvdata(pdev);

	device_remove_file(&pdev->dev, &dev_attr_reg0);
	clk_disable_unprepare(priv->clk);
	return 0;
}

static const struct of_device_id mydrv_of_match[] = {
	{ .compatible = "mycompany,mydrv" },
	{ /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, mydrv_of_match);

static struct platform_driver mydrv_driver = {
	.probe = mydrv_probe,
	.remove = mydrv_remove,
	.driver = {
		.name = "mydrv",
		.of_match_table = mydrv_of_match,
	},
};
module_platform_driver(mydrv_driver);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Platform driver template");
```

!!! note
    `.remove` returns `int` in 5.10 and 6.1 kernels. From 6.11 on, platform drivers use `.remove` returning `void` (temporarily `.remove_new` in 6.3–6.10). Adjust to the kernel version.

Matching device tree node (see [Device tree](device-tree.md)). The address, IRQ, clock and reset values here are placeholders: take them from the hardware manual or from the SoC `.dtsi` node of the peripheral.

```dts
/ {
    mydrv@10049c00 {
        compatible = "mycompany,mydrv";
        reg = <0 0x10049c00 0 0x400>;
        interrupts = <GIC_SPI <irq> IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&cpg CPG_MOD <clk-id>>;
        resets = <&cpg <reset-id>>;
        power-domains = <&cpg>;
        status = "okay";
    };
};
```

On the board, after loading:

```bash
dmesg | grep mydrv
cat /sys/bus/platform/devices/10049c00.mydrv/reg0
```

## Build the module in Yocto

Based on poky's `meta-skeleton/recipes-kernel/hello-mod`. Layout in my layer:

```
recipes-kernel/hello-mod/
├── hello-mod_1.0.bb
└── files/
    ├── Makefile
    └── hello.c
```

The `Makefile` must use `KERNEL_SRC` and provide `modules_install`:

```make
obj-m := hello.o

SRC := $(shell pwd)

all:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC)

modules_install:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules_install

clean:
	rm -f *.o *~ core .depend .*.cmd *.ko *.mod.c *.mod
	rm -f Module.markers Module.symvers modules.order
	rm -rf .tmp_versions Modules.symvers
```

`hello-mod_1.0.bb`:

=== "Dunfell"

    ```
    SUMMARY = "Hello world kernel module"
    LICENSE = "GPL-2.0"
    LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/GPL-2.0;md5=801f80980d171dd6425610833a22dbe6"

    inherit module

    SRC_URI = "file://Makefile file://hello.c"
    S = "${WORKDIR}"

    RPROVIDES_${PN} += "kernel-module-hello"
    ```

=== "Kirkstone and later"

    ```
    SUMMARY = "Hello world kernel module"
    LICENSE = "GPL-2.0-only"
    LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/GPL-2.0-only;md5=801f80980d171dd6425610833a22dbe6"

    inherit module

    SRC_URI = "file://Makefile file://hello.c"
    S = "${WORKDIR}"

    RPROVIDES:${PN} += "kernel-module-hello"
    ```

Add to the image with `IMAGE_INSTALL_append = " hello-mod"` (Kirkstone+: `IMAGE_INSTALL:append`), and autoload it with `KERNEL_MODULE_AUTOLOAD += "hello"` in the recipe.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `insmod: ERROR: could not insert module: Invalid module format` | Built against a different kernel. Compare `modinfo hello.ko \| grep vermagic` with `uname -r` |
| `Unknown symbol in module` | Missing dependency module or symbol not exported (`EXPORT_SYMBOL_GPL`). Check `dmesg` for the symbol name |
| `No rule to make target 'modules'` / `Kernel configuration is invalid` | `KDIR` points to an unconfigured or unbuilt tree. Run `make defconfig && make modules_prepare` in it |
| Driver loads but `probe` never runs | `compatible` mismatch, or node `status` not `okay`. Compare with `/proc/device-tree` |
| `module verification failed: signature and/or required key missing - tainting kernel` | Warning only, the module still loads |

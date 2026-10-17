# BitBake commands

Commands I use daily inside the build environment (`source poky/oe-init-build-env build`).

## Variables

```bash
export RECIPE=linux-renesas     # recipe to work on
export IMAGE=core-image-minimal
```

Recipe names in the Renesas BSP:

| Component | Recipe |
| --------- | ------ |
| Kernel | `linux-renesas` (also reachable as `virtual/kernel`) |
| U-Boot | `u-boot` |
| TF-A (BL2, BL31, FIP) | `trusted-firmware-a` |
| Flash Writer | `flash-writer` |

!!! note "Override syntax: Dunfell vs. Kirkstone and later"
    Dunfell (Yocto 3.1, used by RZ/G2L VLP 3.0.x) uses an underscore: `IMAGE_INSTALL_append`, `SRC_URI_append`, `FILESEXTRAPATHS_prepend`.
    Kirkstone (4.0) and later use a colon: `IMAGE_INSTALL:append`, `SRC_URI:append`, `FILESEXTRAPATHS:prepend`.
    Using the wrong one is silently ignored or raises a parse error, so match the BSP's Yocto version.

## Build and rebuild

Build one recipe (and its dependencies):

```bash
bitbake $RECIPE
```

Force a task to run again even if BitBake thinks it is up to date:

```bash
bitbake $RECIPE -c compile -f
bitbake $RECIPE
```

Re-run from a task onward (capital `-C` invalidates the task and everything after it):

```bash
bitbake $RECIPE -C compile
```

Clean levels, from light to heavy:

```bash
bitbake $RECIPE -c clean          # remove work dir; sstate still used
bitbake $RECIPE -c cleansstate    # also remove sstate -> real rebuild from source
bitbake $RECIPE -c cleanall       # also remove downloaded source -> refetch
```

Keep going after an error, to see all failing recipes at once:

```bash
bitbake -k $IMAGE
```

List the tasks of a recipe:

```bash
bitbake $RECIPE -c listtasks
```

## Inspect variables

Print the final value of a variable after all `.bbappend` and overrides:

```bash
bitbake -e $RECIPE | grep ^SRC_URI=
bitbake -e $RECIPE | grep ^WORKDIR=
bitbake -e $IMAGE  | grep ^IMAGE_INSTALL=
```

Show where a variable was set or changed (history):

```bash
bitbake -e $RECIPE | less     # search for "# $SRC_URI" to see every assignment
```

## Find things

Which layer provides a recipe, and which version:

```bash
bitbake-layers show-recipes "$RECIPE"
bitbake-layers show-recipes "*u-boot*"
```

All `.bbappend` files applied to recipes:

```bash
bitbake-layers show-appends | grep -A5 $RECIPE
```

Layers and their priorities:

```bash
bitbake-layers show-layers
```

Find the recipe file on disk:

```bash
find .. -name "${RECIPE}*.bb" -not -path "*/tmp/*"
```

## Packages and files

Which package installs a file (needs a finished build):

```bash
oe-pkgdata-util find-path /usr/sbin/i2cdetect
```

Which recipe produces a package, and what files a package contains:

```bash
oe-pkgdata-util lookup-recipe i2c-tools
oe-pkgdata-util list-pkg-files i2c-tools
```

Add packages to the image, in `conf/local.conf`:

=== "Dunfell"

    ```
    IMAGE_INSTALL_append = " i2c-tools can-utils devmem2 ethtool"
    ```

=== "Kirkstone and later"

    ```
    IMAGE_INSTALL:append = " i2c-tools can-utils devmem2 ethtool"
    ```

!!! warning
    Keep the leading space inside the quotes. `_append` / `:append` does not add a separator.

## Logs and work directory

Each recipe builds in its own `WORKDIR`:

```bash
WORKDIR=$(bitbake -e $RECIPE | sed -n 's/^WORKDIR="\(.*\)"/\1/p'); echo $WORKDIR
```

Task logs and the exact scripts that ran:

```bash
ls $WORKDIR/temp/
less $WORKDIR/temp/log.do_compile
less $WORKDIR/temp/run.do_compile
```

When a task fails, BitBake prints the log path at the end of the error. That log is the first thing to read.

## Interactive shell

Open a shell with the recipe's cross-compile environment, in the source directory:

```bash
bitbake $RECIPE -c devshell
```

Useful for trying a `make` command manually. Changes made here are lost on the next `clean`, so use `devtool` for real changes.

## Kernel-specific tasks

```bash
bitbake virtual/kernel -c menuconfig      # interactive config
bitbake virtual/kernel -c diffconfig      # write changes as a config fragment
bitbake virtual/kernel -c savedefconfig   # write a minimal defconfig
bitbake virtual/kernel -c kernel_configcheck -f
```

`diffconfig` and `savedefconfig` print the output path. Use the fragment in a `.bbappend` to keep the change.

## Dependencies

Why is a package in my image? Generate dependency files and search them:

```bash
bitbake -g $IMAGE
grep '"i2c-tools"' task-depends.dot | head
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| Change in a `.bbappend` has no effect | Wrong override syntax for the Yocto version, or layer not in `bblayers.conf`. Verify with `bitbake -e` |
| `Taskhash mismatch` / `basehash` errors | Corrupt sstate or a non-deterministic recipe. `bitbake $RECIPE -c cleansstate` and rebuild |
| `ERROR: Unable to start bitbake server` | A previous BitBake is still running or crashed. Wait, or remove `bitbake.lock` in the build dir |
| Rebuilt the kernel but the image has the old one | Rebuild the image too: `bitbake $IMAGE` |

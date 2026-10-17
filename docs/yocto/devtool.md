# devtool

`devtool` checks out a recipe's source into a git workspace, builds the recipe from there, and at the end turns my commits into patches inside my layer. This is how I modify the kernel or U-Boot within Yocto.

## Variables

```bash
export BSP_DIR=~/yocto/rzg2l_vlp_v3.0.x
export RECIPE=linux-renesas          # or u-boot, trusted-firmware-a
export LAYER=../meta-mylayer         # destination layer, relative to build/
export MACHINE=smarc-rzg2l
export BOARD_IP=192.168.1.5
```

```bash
cd $BSP_DIR && source poky/oe-init-build-env build
```

## Workflow overview

```
devtool modify  ->  edit + git commit  ->  bitbake / devtool build  ->  test on board
       ^                                                                     |
       +----------------------------- repeat --------------------------------+
                                       |
                                devtool finish  ->  patches + .bbappend in my layer
```

## 1. Check out the source

```bash
devtool modify $RECIPE
```

The source (with all BSP patches already applied) is now a git repository at:

```bash
cd $BSP_DIR/build/workspace/sources/$RECIPE
git log --oneline -5
git branch        # work happens on branch "devtool"
```

From now on BitBake builds this recipe from the workspace, not from the original source.

## 2. Edit and commit

Edit files normally, then commit. Each commit becomes one patch:

```bash
cd $BSP_DIR/build/workspace/sources/$RECIPE
git add -A
git commit -s -m "arm64: dts: renesas: rzg2l-smarc: enable SCI1"
```

!!! tip
    Use kernel-style commit subjects (`subsystem: component: what`). They become the patch file names, e.g. `0001-arm64-dts-renesas-rzg2l-smarc-enable-SCI1.patch`.

## 3. Build

```bash
cd $BSP_DIR/build
bitbake $RECIPE
```

For the kernel, this also runs `do_deploy`, so the new `Image` and dtb land in `tmp/deploy/images/$MACHINE/`. Copy them to TFTP for a quick test:

```bash
sudo cp -L tmp/deploy/images/$MACHINE/Image tmp/deploy/images/$MACHINE/*.dtb /srv/tftp/
```

!!! note
    `devtool build $RECIPE` also works, but stops before `do_deploy`. For the kernel, `bitbake $RECIPE` is more convenient because the output ends up in the deploy directory.

### Kernel config through devtool

```bash
devtool menuconfig linux-renesas
```

The changes are saved as `oe-local-files/devtool-fragment.cfg` in the workspace and included by `devtool finish`.

### Applications: deploy directly to the board

For user-space recipes, install the build result on a running board over SSH:

```bash
devtool build hello
devtool deploy-target hello root@$BOARD_IP
# remove it again
devtool undeploy-target hello root@$BOARD_IP
```

## 4. Finish: write patches into my layer

```bash
devtool finish $RECIPE $LAYER
```

This:

- writes each commit as a patch to `$LAYER/recipes-.../<recipe>/files/` (or a version-named directory),
- creates or updates the `.bbappend` with `SRC_URI` entries for the patches,
- removes the recipe from the workspace, so BitBake builds from the original source + patches again.

Check what was written:

```bash
cd $LAYER && git status && git diff
```

!!! warning
    `devtool finish` fails if the workspace has uncommitted changes. Commit or `git stash` first.

## Other useful commands

```bash
devtool status                         # recipes currently in the workspace
devtool update-recipe -a $LAYER $RECIPE   # write patches but keep working in the workspace
devtool reset $RECIPE                  # drop the workspace, discard changes (source dir is kept)
devtool add hello https://github.com/<user>/hello.git    # new recipe from a git repo
devtool upgrade $RECIPE -V <version>   # move a recipe to a newer upstream version
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `ERROR: recipe xxx is already in your workspace` | A previous `devtool modify` was never finished. `devtool reset $RECIPE`, or keep working in the existing workspace |
| `Source tree path ... already exists and is not empty` after reset | `devtool reset` keeps the source. Delete `workspace/sources/$RECIPE` or use `devtool modify $RECIPE <other-dir>` |
| Changes not in the build | Not committed? BitBake builds the working tree, but `finish` only exports commits. Also check `devtool status` |
| Patches written to the wrong layer | Pass the layer path explicitly to `devtool finish`, and check the layer is in `bblayers.conf` |

# Patches

Creating, checking and applying patches for the kernel (same commands for U-Boot and TF-A trees).

## Variables

```bash
export KSRC=~/work/rz_linux-cip
export BASE=<base-commit>          # commit my work starts from, e.g. the Yocto SRCREV
export OUT=~/work/patches
```

## Commit in kernel style

```bash
cd $KSRC
git add -p
git commit -s
```

Message format:

```
arm64: dts: renesas: rzg2l-smarc: Enable SPI1

Enable RSPI1 on the PMOD connector and add the pin configuration.

Signed-off-by: Nhat Tran <you@example.com>
```

- Subject: `subsystem: component: Short description`, under ~72 characters. Look at how others did it: `git log --oneline -- <file>`.
- `-s` adds the `Signed-off-by` line.

Fix the last commit:

```bash
git commit --amend
```

Fix an older commit in the series:

```bash
git commit --fixup=<commit>
git rebase -i --autosquash $BASE
```

## Create patches

All commits since `BASE`, one file per commit:

```bash
rm -rf $OUT && git format-patch $BASE -o $OUT
ls $OUT
```

```
0001-arm64-dts-renesas-rzg2l-smarc-Enable-SPI1.patch
0002-spi-rspi-Fix-xxx.patch
```

Only the last N commits:

```bash
git format-patch -3 -o $OUT
```

A second version of a series, with a cover letter:

```bash
git format-patch -v2 --cover-letter $BASE -o $OUT
```

## Check patches

Style check (the kernel's own tool):

```bash
./scripts/checkpatch.pl --strict $OUT/*.patch
```

Check a commit directly, without creating files:

```bash
./scripts/checkpatch.pl --strict -g HEAD
```

Who to send the patch to (upstream submissions):

```bash
./scripts/get_maintainer.pl $OUT/0001-*.patch
```

## Apply patches

With git (keeps author, message and date):

```bash
git am $OUT/*.patch
```

Test whether patches apply, without changing anything:

```bash
git apply --check $OUT/*.patch
```

Without git, e.g. in an extracted source tarball:

```bash
patch -p1 --dry-run < 0001-xxx.patch
patch -p1 < 0001-xxx.patch
```

## When `git am` fails

```bash
git am --3way $OUT/*.patch        # retry using the common ancestor; often resolves context drift
```

If it still stops with a conflict:

```bash
git status                        # see conflicting files
# edit and fix conflict markers
git add <fixed files>
git am --continue
```

Other options:

```bash
git am --skip                     # drop the current patch
git am --abort                    # go back to the state before git am
```

Apply what can be applied and leave `.rej` files for the rest:

```bash
git apply --reject $OUT/0002-*.patch
find . -name "*.rej"
```

## Compare two versions of a series

Shows what changed between v1 and v2, commit by commit:

```bash
git range-diff $BASE..work-v1 $BASE..work-v2
```

## Move patches into Yocto

Copy them into my layer and list them in the `.bbappend` (see [Custom layer & bbappend](../yocto/custom-layer.md)):

```bash
cp $OUT/*.patch $BSP_DIR/meta-mylayer/recipes-kernel/linux/files/
```

Generate the `SRC_URI` lines:

```bash
for p in $OUT/*.patch; do echo "    file://$(basename $p) \\"; done
```

!!! tip
    Patches must apply on the exact source the recipe uses (`SRCREV` plus the BSP's own patches). Making them with [devtool](../yocto/devtool.md) guarantees that.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `patch does not apply` | Base differs. Use `git am --3way`, or rebase my commits onto the target base and regenerate |
| `Patch is empty` | Commit already exists in the tree. `git am --skip` |
| `git am` reports `previous rebase directory .git/rebase-apply still exists` | A previous `git am` was not finished. `git am --abort` |
| Yocto `do_patch` fails after BSP upgrade | Kernel moved on. `devtool modify`, re-apply with `git am --3way`, then `devtool finish` |

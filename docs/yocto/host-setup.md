# Host setup

Preparing an Ubuntu machine (native or VM) to build the RZ BSP.

## Requirements

| Item | Recommended |
| ---- | ----------- |
| OS | Ubuntu 20.04 for Dunfell-based BSPs (RZ/G2L VLP 3.0.x), Ubuntu 22.04 for Kirkstone/Scarthgap-based BSPs |
| Disk | 150 GB free or more (a `core-image-weston` build with SDK uses ~100 GB) |
| RAM | 16 GB or more, plus swap |
| CPU | As many cores as possible; build time scales almost linearly |

!!! tip
    The supported host distro is listed in the release note of each BSP. When the host is newer than supported, BitBake prints a warning (`Host distribution "ubuntu-24.04" has not been validated`). It usually still works, but use the supported version, or a container (see below), when you hit strange host-tool errors.

## Install packages

```bash
sudo apt-get update
sudo apt-get install -y gawk wget git diffstat unzip texinfo gcc build-essential \
    chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils \
    iputils-ping python3-git python3-jinja2 python3-subunit zstd liblz4-tool file \
    locales libacl1 libsdl1.2-dev xterm p7zip-full libyaml-dev libssl-dev bmap-tools
```

!!! note
    On Ubuntu 20.04, `liblz4-tool` provides `lz4c`; on 22.04 and later the package is `lz4`. Install whichever exists.

## Locale

BitBake requires a UTF-8 locale:

```bash
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
```

## Git identity

Some recipes apply patches with `git am`, which fails without an identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

## Increase inotify limits

Avoids `No space left on device` errors from file watchers (the disk is not actually full) during large builds:

```bash
echo "fs.inotify.max_user_watches=524288" | sudo tee /etc/sysctl.d/99-yocto.conf
sudo sysctl --system
```

## Shared download and sstate directories

Keep downloads and the shared state cache outside any single build tree, so several BSP builds can reuse them:

```bash
mkdir -p ~/yocto/downloads ~/yocto/sstate-cache
```

These directories are referenced from `local.conf` when building the BSP.

## Optional: build inside a container

When the host Ubuntu version differs from what the BSP supports, build in a container of the supported version:

```bash
docker run --rm -it \
    -v ~/yocto:/home/build/yocto \
    -w /home/build/yocto \
    crops/poky:ubuntu-20.04
```

`crops/poky` images already contain all required packages and run as a non-root user matching the mounted directory owner.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `Please use a locale setting which supports UTF-8` | Run the [Locale](#locale) step and open a new shell |
| `Do not use Bitbake as root` | Build as a normal user, never with `sudo` |
| `Host distribution ... has not been validated` | Warning only; use a container if the build fails in host tools |
| Build killed randomly / `c++: fatal error: Killed signal` | Out of memory. Lower `BB_NUMBER_THREADS` / `PARALLEL_MAKE` or add swap |
| `No space left on device` with free disk | Raise inotify limits, or check `df -i` for exhausted inodes |

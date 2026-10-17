# SDK

The SDK is a self-contained cross toolchain plus a sysroot matching the image. With it I can build applications, kernel modules and even the kernel without running BitBake.

## Variables

```bash
export BSP_DIR=~/yocto/rzg2l_vlp_v3.0.x
export IMAGE=core-image-minimal
export SDK_DIR=/opt/poky/3.1.x           # install location; one directory per SDK
export BOARD_IP=192.168.1.5
```

## Build the SDK installer

Build the SDK from the **same image** that runs on the board, so the libraries match:

```bash
cd $BSP_DIR && source poky/oe-init-build-env build
bitbake $IMAGE -c populate_sdk
ls tmp/deploy/sdk/
```

```
poky-glibc-x86_64-core-image-minimal-aarch64-smarc-rzg2l-toolchain-3.1.x.sh
```

### Extra content in the SDK

Add to `conf/local.conf` before building the SDK, if needed:

=== "Dunfell"

    ```
    # kernel source for building out-of-tree modules
    TOOLCHAIN_TARGET_TASK_append = " kernel-devsrc"
    # cmake on the host side
    TOOLCHAIN_HOST_TASK_append = " nativesdk-cmake"
    ```

=== "Kirkstone and later"

    ```
    TOOLCHAIN_TARGET_TASK:append = " kernel-devsrc"
    TOOLCHAIN_HOST_TASK:append = " nativesdk-cmake"
    ```

## Install

```bash
sudo sh tmp/deploy/sdk/poky-glibc-x86_64-$IMAGE-*-toolchain-*.sh -d $SDK_DIR -y
```

!!! tip
    Install SDKs for different BSPs or boards into different directories (`/opt/poky/rzg2l-3.1.x`, `/opt/poky/rzv2n-5.0`, ...). Sourcing two environments in the same shell mixes up the toolchains.

## Use the SDK

Source the environment in each new shell:

```bash
source $SDK_DIR/environment-setup-aarch64-poky-linux
```

Check:

```bash
echo $CC
echo $CROSS_COMPILE $ARCH
$CC --version
```

```
aarch64-poky-linux-gcc -mcpu=cortex-a55 ... --sysroot=/opt/poky/3.1.x/sysroots/aarch64-poky-linux
aarch64-poky-linux- arm64
```

`$CC` already contains `--sysroot` and CPU flags. Always use `$CC`, never `aarch64-poky-linux-gcc` alone.

## Build an application

Single file:

```bash
$CC hello.c -o hello
file hello
```

```
hello: ELF 64-bit LSB pie executable, ARM aarch64, ...
```

Makefile project: the SDK exports `CC`, `CXX`, `CFLAGS`, `LDFLAGS`, so a Makefile that uses these variables works as is:

```make
CFLAGS += -Wall -O2

app: main.o util.o
	$(CC) $(LDFLAGS) $^ -o $@

clean:
	rm -f app *.o
```

```bash
make
```

!!! warning
    Do not hardcode `CC = gcc` in the Makefile. It overrides the SDK's cross compiler and builds for the host.

CMake project (requires `nativesdk-cmake` in the SDK; the environment script then makes `cmake` use the SDK toolchain file):

```bash
cmake -S . -B build
cmake --build build -j$(nproc)
```

Autotools project:

```bash
./configure $CONFIGURE_FLAGS
make -j$(nproc)
```

## Copy to the board and run

```bash
scp hello root@$BOARD_IP:/home/root/
ssh root@$BOARD_IP /home/root/hello
```

With NFS boot, copy straight into the exported rootfs instead:

```bash
sudo cp hello /nfs/rzg2l/home/root/
```

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `cannot execute binary file: Exec format error` | Running an ARM binary on the host, or a host binary on the board. Check with `file` |
| `version 'GLIBC_2.xx' not found` on the board | SDK built from a different image/BSP than the rootfs. Rebuild the SDK from the running image |
| `fatal error: xxx.h: No such file or directory` | Library's `-dev` package not in the SDK. Add the library to the image and rebuild the SDK |
| `aarch64-poky-linux-ld: unrecognized option '-Wl,-O1'` when building the kernel | The SDK's `LDFLAGS` are for gcc, not ld. Run `unset LDFLAGS` before `make` in a kernel tree |
| CMake builds for x86 | `nativesdk-cmake` missing from the SDK, or the system `cmake` was used. Check `which cmake` |

---
title: "Working on JITLink"
date: '2023-11-26'
draft: false
summary: 'In this post, I talk about the initial steps of contributing to LLVM
JITLink project.'
tags:
  - llvm
  - linking
  - arm
  - just-in-time
image:
author: Manas
---

This post contains components for getting _up and running_ with contributing to
the [LLVM JITLink](https://llvm.org/docs/JITLink.html#jit-linking) project.
Especially if you cannot get a hands on a Raspberry Pi, you can still go ahead
(albeit, with some limitations) by emulating the RPi in QEMU.


## Running RPi in QEMU

This [repository](https://github.com/dhruvvyas90/qemu-rpi-kernel) contains
information about emulation. There are two ways to emulate RPi in QEMU.

1. Using pre-built kernels and `dtb` (device tree) files 

[devicetree](https://en.wikipedia.org/wiki/Devicetree) is a mechanism to provide
a way to describe _non-discoverable hardware_.
The repository consists of various pre-built kernels and device tree files which
can directly boot an RPi image. Though a pretty straightforward way to emulate,
it has a caveat which has to do due to an error in the provided devicetree
files. [This issue](https://github.com/dhruvvyas90/qemu-rpi-kernel/issues/82)
explains how this error limits CPU usage to `1 core` and limits RAM to `256MB`.
This may be a serious bottleneck for you.

To tackle this limitation, you can otherwise use `native emulation`, which is a
direct backend support from QEMU to emulate `raspi2b/3b` (and few other boards).

2. Native emulation

QEMU natively provides support for certain [RPi
boards](https://www.qemu.org/docs/master/system/arm/raspi.html). This backend is
used along with the pre-built kernel and dtb file. And the image is then run by
the QEMU backend. Some of the board backends provide upto 4 cores and 1GB of
RAM.
The latest image
([bookworm](https://downloads.raspberrypi.com/raspios_armhf/images/raspios_armhf-2023-10-10/2023-10-10-raspios-bookworm-armhf.img.xz))
will not work for native emulation, use the legacy image
([bullseye](https://downloads.raspberrypi.com/raspios_oldstable_armhf/images/raspios_oldstable_armhf-2023-10-10/2023-05-03-raspios-bullseye-armhf.img.xz)).

<u>Note</u>: You can cross-compile your own kernel and boot that instead (in
either of the two ways above). But you should use correct devicetree file for
corresponding kernel.

On a sidenote, if you want to edit the raspios image file, you can mount it
as a loop device. I needed it for once changing the default password and locale.
The image contains two partitions, first contains the bootloader, second is the
Linux filesystem. The bootloader contains kernel and various copies of
devicetree files.


```bash
$ fdisk -l 2023-05-03-raspios-bullseye-armhf-lite.img
Device                                      Boot  Start     End Sectors  Size Id Type
2023-05-03-raspios-bullseye-armhf-lite.img1        8192  532479  524288  256M  c W95 FA
2023-05-03-raspios-bullseye-armhf-lite.img2      532480 3842047 3309568  1.6G 83 Linux

$ sudo losetup -f --show -P <img-file>
/dev/loopN
$ sudo mount /dev/loopNp1 /mnt/     # mounts FAT32
$ sudo mount /dev/loopNp2 /mnt/     # mounts the linux fs
... changes to the image ...
$ sudo umount /mnt/
$ sudo losetup -d /dev/loopN        # unmount and detach loop device

```


If you are facing issues with a network connection during native-emulation,
check if a network card is available or not via `$ ip a`. If there is no network
card, you must tell QEMU to emulate it.

```bash
-device usb-net,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22
```

The QEMU command will be:

```bash
qemu-system-aarch64 -m 1024 -M raspi3b -nographic \
    -kernel kernel8.img -dtb bcm2710-rpi-3-b-plus.dtb -sd <raspios.img> \
    -append "console=ttyAMA0 root=/dev/mmcblk0p2 rw rootwait rootfstype=ext4" \
    -device usb-net,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22
```

## Cross-compiling JITLink for AArch32 (on Fedora)

Another way to reduce build efforts can be to cross-compile the `llvm-jitlink`
and `llvm-jitlink-executor` binaries on your host machine. To cross-compile
JITLink, we require `arm-linux-gnueabihf-gcc` compiler and its corresponding
glibc, and although Fedora officially provides one, that compiler is only
able to build kernels and no userspace programs can be
built[[1]](https://packages.fedoraproject.org/pkgs/cross-gcc/gcc-arm-linux-gnu/).

But fortunately, a [copr
package](https://copr.fedorainfracloud.org/coprs/lantw44/arm-linux-gnueabihf-toolchain/)
is available which provides a latest build of the toolchain.

You need to enable it and then you can install the compiler.

```bash
$ sudo dnf copr enable lantw44/arm-linux-gnueabihf-toolchain
$ sudo dnf install arm-linux-gnueabihf-{gcc,glibc}
```

You would also require a host build of `llvm-tblgen`. You can cross compile
JITLink using this cmake command:

```bash
cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CROSSCOMPILING=1 \
  -DCMAKE_C_COMPILER=/usr/bin/arm-linux-gnueabihf-gcc \
  -DCMAKE_CXX_COMPILER=/usr/bin/arm-linux-gnueabihf-g++ \
  -DLLVM_HOST_TRIPLE=arm-unknown-linux-gnueabihf \
  -DLLVM_TABLEGEN=<path/to/host/build>/bin/llvm-tblgen \
  -DLLVM_ENABLE_PIC=OFF \
  -DLLVM_BUILD_STATIC=ON \
  -DLLVM_ENABLE_LIBXML2=OFF \
  <path/to/llvm>
```

You must verify that the gcc and glibc version on host machine matches on the
user RPi machine (either the emulated RPi or an actual one). If not, then you
will have to build appropriate version of glibc which can be another hassle.

Once built, you can copy these binaries over to the RPi via `scp` or similar
tools.

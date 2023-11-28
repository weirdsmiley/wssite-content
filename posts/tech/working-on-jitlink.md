---
title: "Working on JITLink"
date: '2023-11-26'
draft: true
summary: 'In this post, I talk about the initial steps of contributing to LLVM
JITLink project.'
tags:
  - llvm
  - linking
  - arm
  - just-in-time
image: https://dcvp84mxptlac.cloudfront.net/diagrams2/MATH7-5-1-X-2.png
author: Manas
---

This post contains components for getting _up and running_ with contributing to
the [LLVM JITLink](https://llvm.org/docs/JITLink.html#jit-linking) project.


Before everything, we must first understand why JITLink is used. Were we not
doing fine with static linking and dynamic loading?


If you cannot get a hands on a Raspberry Pi, you can still work (albeit, with
some limitations) by **emulating** the RPi in QEMU.

## Running RPi in QEMU

Following this [repository](https://github.com/dhruvvyas90/qemu-rpi-kernel),
there are two ways to emulate RPi in QEMU.

1. Using pre-built kernels and `dtb` (device tree) files 

- [devicetree](https://en.wikipedia.org/wiki/Devicetree) provides a way to
     describe _non-discoverable hardware_.

The repository consists of various pre-built kernels and device tree files which
can directly boot an RPi image. Though a pretty straightforward way to emulate,
it has a major caveat which has to do due to an error in the provided dtb files.
[This issue](https://github.com/dhruvvyas90/qemu-rpi-kernel/issues/82) explains
how this error limits CPU usage to `1 core` and limits RAM to `256MB`. This may
be a serious bottleneck for you.

To tackle this limitation, you can otherwise use `native emulation`, which is a
direct backend support from QEMU to emulate `raspi2b/3b` (and few other boards).

2. Native emulation
   - QEMU natively provides support for certain
     [RPi boards](https://www.qemu.org/docs/master/system/arm/raspi.html).
     Here this backend is used along with the above mentioned pre-built kernel
     and dtb file. Both these files are placed inside the image file. And the
     image is then run by the QEMU backend.
   - Some of the board backends provide upto 4 cores and 1GB of RAM.

- Note: You can cross-compile your own kernel and boot that instead (in either
  of the above two ways). But you should use correct dtb file for corresponding
  kernel. I don't have a map between kernels and dtb files.

- If you want to edit the raspios image file, you can mount it as a loop
  device. I needed it for once changing the default password and locale.
  The image contains two partitions, first contains the FAT32 bootloader,
  second contains the Linux fs. The bootloader contains kernel and various
  copies of dtb files.
- The latest image (bookworm) isn't working, use the legacy image (bullseye).

```bash
Disk 2023-10-10-raspios-bookworm-arm64-lite.img: 2.54 GiB, 2722103296 bytes, 5316608 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x7788c428

Device                                      Boot   Start     End Sectors  Size Id Type
2023-10-10-raspios-bookworm-arm64-lite.img1         8192 1056767 1048576  512M  c W95 FAT32 (LBA)
2023-10-10-raspios-bookworm-arm64-lite.img2      1056768 5316607 4259840    2G 83 Linux
```

```bash
$ sudo losetup -f --show -P <img-file>
/dev/loopN
$ sudo mount /dev/loopNp1 /mnt/     # mounts FAT32
$ sudo mount /dev/loopNp2 /mnt/     # mounts the linux fs
... changes to the image ...
$ sudo umount /mnt/
$ sudo losetup -d /dev/loopN        # unmount and detach loop device

```


**`TODO`**

I used native emulation. And now my goal is to test JITLink tests on it,
natively that is both executor and connector on the same machine, and via host,
which means executor will be on the qemu and connector on the host machine.
Unable to proceed because of no network interface on bullseye qemu (was able to
boot via native-emulation).

## Cross-compiling JITLink for AArch32 (on Arch, Fedora)

* [arch package](https://archlinux.org/packages/extra/x86_64/arm-none-eabi-gcc/)

* using `dnf copr` lantw4?? packages
  * Check out my cross-compilation cmake build script
  * 
* rpi
  * https://askubuntu.com/a/1090440

## JITLink in GDB

So I ran the `llvm-jitlink` via gdb and th executor on my RPi to gather the
call info. The main resides in `llvm-jitlink.cpp` which first initializes
targets, MCs and disassemblers, so that they can be used if needed. It would
depend on what the user wants to do. Then it _sanitizes_ the arguments passed
via cmdline. This involves finding out what necessary steps are asked to be
taken, for example, connect to the executor via TCP and execute the program. It
also initializes a `TIMER` called JITLinkTimers, which I reckon is symbolically
the clock on SoC.

It then figures out the target triple and features from the provided object
file, and creates a new Session to be run. It figures out the EntryPoint which
would be main in our case, and its address. And depending upon the
initialization of OrcRuntime, it either runs the program with runtime or
without runtime. As it figured out that we want `--oop-executor-connect` it
creates a `SimpleRemoteEPC` (a remote executor) which connects to the listening
executor on RPi via a TCP connection. This class implements a `runAsMain`
function which takes in the EntryPoint and its address. Following, it calls
wrapper functions which can pass the address to remote and let it handle. These
wrapper functions use _promise_ to collect the result in a buffer (called
`ResultBuffer`).

**`TODO`** Write-up of how the executor takes in the fn addr and executes it.

## JITLink and TableGen

* From JITLink Sync meeting minutes
  * generate information via tablegen
    - what kind of information?
      - FixupInfo : [[file:///home/manas/workspace/llvm/llvm-project.git/jitlink/llvm/include/llvm/ExecutionEngine/JITLink/aarch32.h]] line 177
    - tablegen generates read-only data. can we simply generate the `static
      constexpr ...` instead of putting them in a file, so that we don't have to
      read a file and embed it in our source, instead we can simply generate
      `static constexpr ...` inside of our source. like how mlir does it. but
      mlir also generates entire source not just some part of it.
    - An [[file:///home/manas/workspace/llvm/llvm-project.git/jitlink/llvm/lib/Target/ARM/ARMInstrInfo.td|example]] tablegen file which ARM uses
    - there shouldn't be any substantial overhead, dynamic linkers are expected
      to be fast

* What do we need to do?
  We are hard-coding FixupInfo (?explain) and our objective here is, can we use
  TableGen to auto-generate this information in a more generic way?
* Caveats right now:
  * TableGen generates entire modules(files) of read-only data, we aren't
    interested in entire files, only the `static constexpr ...` and that too
    not inside a file, but possibly in-memory, so can we can simple copy the
    data from memory to our source files during compilation. This is very
    similar to header pre-processing. We have header files and when the
    pre-processors is initialized during first phase of compilation, it simply
    copies the appropriate headers in respective .c/.cpp files.
  * Now first, we need to figure out how tablegen files are used to produce
    read-only data. This will help us in designing a build procedure for our
    jitlink compilation pipeline.
  * We also need to document the entire FixupInfo in a more generic manner, for
    which we refer to the armv7/8 abi spec. What those hex values mean must be
    documented. How the RISCV folks have segragated their ISA in R/S/J-class of
    instructions, we need a similar approach here.
* Approaches:
  * A complete usage is: here they use tablegen to generate `constexpr` and
    including the generated data in their sources.
    * [[file:///home/manas/workspace/llvm/llvm-project.git/build.mlir/include/llvm/Frontend/OpenACC/ACC.h.inc]]
    * [[file:///home/manas/workspace/llvm/llvm-project.git/jitlink/llvm/unittests/Frontend/OpenACCTest.cpp]]
  * We write a .td file which contains recipes for codegen, `llvm-tblgen`
    parses our information and constructs `records` out of it. The backend is
    responsible to interpret the constructed records.
    * Which backend should be suitable for our purposes?
    * Many backends use `#ifdef` conditions which help the backend in
      identifying which output format the data is requested. For example, a
      tablegen file is parsed to be a cpp class, we can add two types of
      conditions `#define GEN_DECL` and `#define GEN_DEF` and the backend will
      use the same generated record, but output either the declaration or the
      definition based on the macro.
      * We may not require a macro as the only output we want from our backend
        is `static constexpr ...`
      * Or maybe if we have multiple uses we can utilize macros in this way. This is a sidenote.
      * Looking at all the
        [backends](https://llvm.org/docs/TableGen/BackEnds.html#llvm-backends),
        CodeEmitter is looking suitable. Or mayber InstrInfo, there isn't
        enough information about the differences between the two. Find! **`TODO`**
      * My task is not to create a simple example following FixupInfo and
        generate `static constexpr ...` using either CodeEmitter or InstrInfo
        backend.

## References

1. [JITLink introduction slides](https://llvm.org/devmtg/2022-11/slides/Tutorial2-JITLink.pdf)
2. [Initial AArch32 backend](https://reviews.llvm.org/D144083)
3. [Relocation and symbol resolution](https://en.wikipedia.org/wiki/Relocation_(computing))
4. [LLVM Dev Mtg: JITLink: Native Windows JITing in LLVM](https://www.youtube.com/watch?v=UwHgCqQ2DDA|)
5. [initial patch](https://reviews.llvm.org/D144083)
6. [EuroLLVM tutorial](https://www.youtube.com/watch?v=9jFXNRzDSf0)
7. [current GOT/PLT patch](https://github.com/weliveindetail/llvm-project/commit/660d46587b04fcc)
8. [official docs](https://llvm.org/docs/JITLink.html)
9. [Eymen's test coverage PR](https://github.com/llvm/llvm-project/pull/69636)
10. [Eymen's GSoC proposal](https://docs.google.com/document/d/1sXoIlNGlOJgzDt0DdKtZckr3GeLeSdXOrDxkbJn5IXA/edit#heading=h.2muqj1leprs4)
11. [some background on debugging](https://weliveindetail.github.io/blog/post/2022/11/27/gdb-jit-interface-101.html)

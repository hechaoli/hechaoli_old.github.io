---
layout: post
title: Boot a Raspberry Pi 4 using u-boot and Initramfs
tags: [Linux, Embedded System, Raspberry Pi]
---

# 0. Overview

This purpose of this post is to understand the four components of embedded
Linux -- toolchain, bootloader, kernel and root filesystem -- by using minimal
code and commands to boot a Raspberry Pi 4 from scratch.

# 1. Hardware Requirements
* A Linux desktop machine to compile source code. I'm using Ubuntu 20.04.
* A Raspberry 4 model b with a power adapter.
* An SD card and card reader. I'm using a 2GB SD card.

# 2. Preparation

The SD card will be used to store the bootloader and the root filesytem. So, we
first create two partitions -- `boot` (in `FAT32` format) and `root` (in `ext4`
format) -- on it.

## 2.1 Find the SD card device name
After inserting the SD card reader to the Linux desktop machine, find its
device name by `dmesg`.
```bash
$ dmesg | tail
[19304.704047] usbcore: registered new interface driver uas
[19305.719653] scsi 33:0:0:0: Direct-Access     Mass     Storage Device   1.00 PQ: 0 ANSI: 0 CCS
[19305.720283] sd 33:0:0:0: Attached scsi generic sg2 type 0
[19305.725987] sd 33:0:0:0: [sdb] 3842048 512-byte logical blocks: (1.97 GB/1.83 GiB)
[19305.728140] sd 33:0:0:0: [sdb] Write Protect is off
[19305.728142] sd 33:0:0:0: [sdb] Mode Sense: 03 00 00 00
[19305.730188] sd 33:0:0:0: [sdb] No Caching mode page found
[19305.730750] sd 33:0:0:0: [sdb] Assuming drive cache: write through
[19305.757769]  sdb: sdb1 sdb2
[19305.788187] sd 33:0:0:0: [sdb] Attached SCSI removable disk
```

In this case, the device name is `sdb` and it already has two partitions `sdb1`
and `sdb2`. I'll delete them and repartition the SD card.

## 2.2 Delete existing partitions
```bash
$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): d
Partition number (1,2, default 2):

Partition 2 has been deleted.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): p
Disk /dev/sdb: 1.85 GiB, 1967128576 bytes, 3842048 sectors
Disk model: Storage Device
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```
## 2.3 Add two partitions
Add a 100MB `boot` partition and a `root` partition of the remaining size.

```bash
$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-3842047, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-3842047, default 3842047): +100M

Created a new partition 1 of type 'Linux' and of size 100 MiB.
Partition #1 contains a vfat signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (2-4, default 2):
First sector (206848-3842047, default 206848):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (206848-3842047, default 3842047):

Created a new partition 2 of type 'Linux' and of size 1.8 GiB.
Partition #2 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help): t
Partition number (1,2, default 2): 1
Hex code (type L to list all codes): b

Changed type of partition 'Linux' to 'W95 FAT32'.

Command (m for help): p
Disk /dev/sdb: 1.85 GiB, 1967128576 bytes, 3842048 sectors
Disk model: Storage Device
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot  Start     End Sectors  Size Id Type
/dev/sdb1         2048  206847  204800  100M  b W95 FAT32
/dev/sdb2       206848 3842047 3635200  1.8G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

```

## 2.4 Format the partitions
```bash
# FAT32 for boot partition
$ sudo mkfs.vfat -F 32 -n boot /dev/sdb1

# ext4 for root partition
$ sudo mkfs.ext4 -L root /dev/sdb2
```
## 2.5 Mount the partitions
Mount both partitions so that we can write to them.
```bash
$ sudo mount /dev/sdb1 /mnt/boot
$ sudo mount /dev/sdb2 /mnt/root
```

# 3. Toolchain
First, we need a toolchain to compile source code to executables running on
Raspberry Pi 4. The toolchain we build consists of 
* a cross-compiler,
* binary utilities like the assembler and the linker and
* some runtime libraries. 

A cross-compiler is needed because we will compile the code that runs on the
Raspberry Pi 4 (ARM) on a Linux desktop machine (X86).

We may build a complete toolchain from scratch following the steps in Linux
From Scratch [2]. But I'll take a short cut and use
[`crosstool-NG`](https://crosstool-ng.github.io/). For more details about the
process of building a toolchain, see the awesome explanation in [this
article](https://crosstool-ng.github.io/docs/toolchain-construction/).

## 3.1 Download crosstool-NG source

```bash
$ git clone https://github.com/crosstool-ng/crosstool-ng
$ cd crosstool-ng/
# Switch to the latest release
$ git checkout crosstool-ng-1.24.0 -b 1.24.0
```

## 3.2 Build and Install crosstool-NG
The full documentation of installing crosstool-NG can be found
[here](https://crosstool-ng.github.io/docs/install/).
```bash
$ ./bootstrap
$ ./configure --prefix=${PWD}
$ make
$ make install
$ export PATH="${PWD}/bin:${PATH}"
```

## 3.3 Configure crosstool-NG
Before using `crosstool-NG` to build the toolchain, we need to configure it
first. The configurator works the same way as configuring the Linux kernel.
```bash
$ ct-ng menuconfig
```

There are also sample configurations we can get by `ct-ng list-samples`
command. We can use one of them and then tune it using `ct-ng menuconfig`. Here
I'll just use `aarch64-rpi4-linux-gnu` without modification.
```bash
# Basic information about this config
$ ct-ng show-aarch64-rpi4-linux-gnu
[G...]   aarch64-rpi4-linux-gnu
    Languages       : C,C++
    OS              : linux-4.20.8
    Binutils        : binutils-2.32
    Compiler        : gcc-8.3.0
    C library       : glibc-2.29
    Debug tools     : gdb-8.2.1
    Companion libs  : expat-2.2.6 gettext-0.19.8.1 gmp-6.1.2 isl-0.20 libiconv-1.15 mpc-1.1.0 mpfr-4.0.2 ncurses-6.1 zlib-1.2.11
    Companion tools :

# Use this config
$ ct-ng aarch64-rpi4-linux-gnu
```

{: .box-note}
**NOTE:** The OS is linux-4.20.8 meaning binaries compiled by the toolchain
should be able to run on any kernel version >= 4.20.8.

## 3.4 Build the toolchain
To build the toolchain, simply run:
```bash
$ ct-ng build
```

{: .box-warning}
**NOTE:** At the time of writing, the above command failed when trying to
download isl lib because the location `isl.gforge.inria.fr` seems to be down. A
workaround can be found
[here](https://github.com/crosstool-ng/crosstool-ng/issues/1625).

By default, the built toolchain is installed at `~/x-tools/aarch64-rpi4-linux-gnu`.

# 4. Bootloader
The bootloader's job is to set up the system to a basic level (e.g. configure
the memory controller so that DRAM is accessible) and load the kernel.
Typically, the boot sequence is:

1. ROM code that is stored on chip runs. It loads the Secondary Program Loader
(SPL) into Static Random Access Memory (SRAM), which doesn't require a memory
controller. An SPL can be a stripped-down version of the full bootloader like
u-boot. It is needed because of the limited SRAM size.
2. The SPL sets up the memory controller so that DRAM can be accessed and does
some other hardware configurations. Then, it loads the full bootloader into
DRAM.
3. The full bootloader then loads the kernel, the Flattened Device Tree (FDT)
and optionally the initial RAM disk (`initramfs`) into DRAM. Once the kernel is
loaded, the bootloader will hand over the control to it.

## 4.1 Download u-boot source
```bash
$ git clone git://git.denx.de/u-boot.git
$ cd u-boot
$ git checkout v2021.10 -b v2021.10
```

## 4.2 Configure u-boot
Because the bootloader is device-specific, we need to configure it before
building it. Similar to `crosstool-NG`, there are several sample/default
configs under `configs/` directory. We can find one for Raspberry Pi 4 at
`configs/rpi_4_defconfig`. Then we only need to run `make rpi_4_defconfig`.
Before that, we also need to set the `CROSS_COMPILE` environment variable.

```bash
$ export PATH=${HOME}/x-tools/aarch64-rpi4-linux-gnu/bin/:$PATH
$ export CROSS_COMPILE=aarch64-rpi4-linux-gnu-
$ make rpi_4_defconfig
```

## 4.3 Build u-boot
```bash
$ make
```

## 4.4 Install u-boot
We only need to copy the `u-boot.bin` binary compiled in the last step into the
`boot` partition on the SD card.
```bash
$ sudo cp u-boot.bin /mnt/boot
```

{: .box-warning}
**NOTE:** Raspberry Pi has its own proprietary bootloader, which is loaded by
the ROM code and is capable of loading the kernel. However, since I'd like to
use the open source `u-boot`, I'll need to configure the Raspberry Pi boot
loader to load `u-boot` and then let `u-boot` load the kernel.

```bash
# Download Raspberry Pi firmware/boot directory
$ svn checkout https://github.com/raspberrypi/firmware/trunk/boot

# Copy Raspberry Pi 4 bootloader into boot partition
$ sudo cp boot/{bootcode.bin,start4.elf} /mnt/boot/

# Let Raspberry Pi 4 bootloader load u-boot
$ cat << EOF > config.txt
enable_uart=1
arm_64bit=1
kernel=u-boot.bin
EOF
$ sudo mv config.txt /mnt/boot/
```

# 5. Kernel
Next, we compile the Linux kernel.
## 5.1 Download the Kernel Source
Though the original Linux kernel should work, using Raspberry Pi's fork of it
is more stable. Also note that **the kernel version must be higher than the
kernel version configured for the toolchain**.
```bash
$ git clone --depth=1 -b rpi-5.10.y https://github.com/raspberrypi/linux.git
$ cd linux
```

## 5.2 Config and Build the Kernel
We just use the default config for Raspberry Pi 4. See
[here](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/)
for Raspberry Pi 4 model b specifications.
```bash
$ make ARCH=arm64 CROSS_COMPILE=aarch64-rpi4-linux-gnu- bcm2711_defconfig
$ make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-rpi4-linux-gnu-
```

## 5.3 Install the kernel and device tree
Now we copy the kernel image and device tree binary (`*.dtb`) into the `boot`
partition on the SD card.
```
$ sudo cp arch/arm64/boot/Image /mnt/boot
$ sudo cp arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb /mnt/boot/
```

# 6. Root filesystem
See [Filesystem Hierarchy
Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.pdf) for more
details about the basic directory layout of a Linux system.

## 6.1 Create Directories
```bash
$ mkdir rootfs
$ cd rootfs
$ mkdir {bin,dev,etc,home,lib64,proc,sbin,sys,tmp,usr,var}
$ mkdir usr/{bin,lib,sbin}
$ mkdir var/log

# Create a symbolink lib pointing to lib64
$ ln -s lib64 lib

$ tree -d
.
├── bin
├── dev
├── etc
├── home
├── lib -> lib64
├── lib64
├── proc
├── sbin
├── sys
├── tmp
├── usr
│   ├── bin
│   ├── lib
│   └── sbin
└── var
    └── log

16 directories

# Change the owner of the directories to be root
# Because current user doesn't exist on target device
$ sudo chown -R root:root *
```

## 6.2 Build and Install Busybox
We'll use Busybox for essential Linux utilities such as shell. So, we need to
install it to the `rootfs` directory just created.

```bash
# Download the source code
$ wget https://busybox.net/downloads/busybox-1.33.2.tar.bz2
$ tar xf busybox-1.33.2.tar.bz2
$ cd busybox-1.33.2/

# Config
$ CROSS_COMPILE=${HOME}/x-tools/aarch64-rpi4-linux-gnu/bin/aarch64-rpi4-linux-gnu-
$ make CROSS_COMPILE="$CROSS_COMPILE" defconfig
# Change the install directory to be the one just created
$ sed -i 's%^CONFIG_PREFIX=.*$%CONFIG_PREFIX="/home/hechaol/rootfs"%' .config

# Build
$ make CROSS_COMPILE="$CROSS_COMPILE"

# Install
# Use sudo because the directory is now owned by root
$ sudo make CROSS_COMPILE="$CROSS_COMPILE" install
```
## 6.3 Install required libraries

Next, we install some shared libraries required by Busybox. We can find those
libraries by the following command:

```bash
$ readelf -a ~/rootfs/bin/busybox | grep -E "(program interpreter)|(Shared library)"
      [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libresolv.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
```
We need to copy these files from the toolchain's `sysroot` directory to the
`rootfs/lib` directory.

```bash
$ export SYSROOT=$(aarch64-rpi4-linux-gnu-gcc -print-sysroot)
$ sudo cp -L ${SYSROOT}/lib64/{ld-linux-aarch64.so.1,libm.so.6,libresolv.so.2,libc.so.6} ~/rootfs/lib64/
```

## 6.4 Create device nodes
Two device nodes are needed by Busybox.
```bash
$ cd ~/rootfs
$ sudo mknod -m 666 dev/null c 1 3
$ sudo mknod -m 600 dev/console c 5 1
```

# 7. Boot the Board

Finally, with all components ready, we can boot the board. There are two
options in terms of the root filesystem. We can either use it as an initramfs
which can mount a real root filesystem later or use it as the permanent root
filesystem directly.

## 7.1 Option 1: Boot with a initramfs

When is an initramfs
needed? According to Linux From Scratch [2], There are only four primary
reasons to have an initramfs in the LFS environment:
* Loading the rootfs from a network.
* Loading it from an LVM logical volume.
* Having an encrypted rootfs where a password is required.
* For the convenience of specifying the rootfs as a LABEL or UUID.

Instead of using a initramfs, we can also put the root filesystem into the
`root` partition on the SD card directly. In that case, we need to configure
the kernel commandline passed from bootloader to kernel.

### 7.1.1 Build an initramfs
An initramfs is a compressed `cpio` archive, which is an old Unix archive
format similar to `tar` and `zip`.

```bash
$ cd ~/rootfs
$ find . | cpio -H newc -ov --owner root:root -F ../initramfs.cpio
$ cd ..
$ gzip initramfs.cpio
$ ~/u-boot/tools/mkimage -A arm64 -O linux -T ramdisk -d initramfs.cpio.gz uRamdisk

# Copy the initramffs to boot partition
$ sudo cp uRamdisk /mnt/boot/
```

### 7.1.2 Configure u-boot
We need to configure the u-boot so that it can pass the correct kernel
commandline and device tree binary to kernel. For simplicity, I'll use the
`Busybox` shell as the `init` program. In real life, if using initramfs, then
the `init` program should be responsible for mounting the permanent root
filesystem.

```bash
$ cat << EOF > boot_cmd.txt
fatload mmc 0:1 \${kernel_addr_r} Image
fatload mmc 0:1 \${ramdisk_addr_r} uRamdisk
setenv bootargs "console=serial0,115200 console=tty1 rdinit=/bin/sh"
booti \${kernel_addr_r} \${ramdisk_addr_r} \${fdt_addr}
EOF
$ ~/u-boot/tools/mkimage -A arm64 -O linux -T script -C none -d boot_cmd.txt boot.scr

# Copy the compiled boot script to boot partition
$ sudo cp boot.scr /mnt/boot/
```
Meaning of the boot commands:
* Load the kernel image from partition 1 (`boot` partition) into memory.
* Load the initramfs from partition 1 (`boot` partition) into memory.
* Set kernel commandline.
* Boot using the given kernel, device tree binary and initramfs.

{: .box-error}
**NOTE:** In the last line, the last argument is `fdt_addr` rather than
`fdt_addr_r` like the other two arguments. At first, I used `fdt_addr_r` and
couldn't boot the board. I realized the error after finding [this
post](https://forums.raspberrypi.com/viewtopic.php?f=98&t=314845) on Raspberry
Pi forum. Also, according to one of the replies, a current U-boot already
inherits the DTB from the firmware, putting its address in `${fdt_addr}`. So we
don't need to load the `dtb` file in u-boot.

### 7.1.3 Boot it!
Finally, all four components are ready. We can now try booting it. The boot
partition now has the following files:
```bash
$ tree /mnt/boot/
/mnt/boot/
├── bcm2711-rpi-4-b.dtb
├── bootcode.bin
├── boot.scr
├── config.txt
├── Image
├── start4.elf
├── uRamdisk
└── u-boot.bin

0 directories, 7 files
```

Now we unmount the partitions and insert the SD card into Raspberry Pi 4.
```bash
$ sudo umount /dev/sdb1
$ sudo umount /dev/sdb2
```

After powering up Raspberry Pi 4, we should get a `Busybox` shell if
successful.

## 7.2 Option 2: Boot with a permanent rootfs directly
Alternatively, we can boot with the `root` partition being the root filesystem
directly. To do so, following the steps below.

### 7.2.1 Copy rootfs to root partition on the SD card
Insert the SD card to the card reader and insert the reader to the Linux
desktop.
```bash
$ sudo mount /dev/sdb1 /mnt/root
$ sudo mount /dev/sdb2 /mnt/root
$ cp -r ~/rootfs/* /mnt/root/
```

### 7.2.2 Change boot commands
We no longer need the initramfs.
```bash
$ cat << EOF > boot_cmd.txt
fatload mmc 0:1 \${kernel_addr_r} Image
setenv bootargs "console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rw rootwait init=/bin/sh"
booti \${kernel_addr_r} - \${fdt_addr}
EOF
$ ~/u-boot/tools/mkimage -A arm64 -O linux -T script -C none -d boot_cmd.txt boot.scr
$ sudo cp boot.scr /mnt/boot/

# Remove the initramfs as it's not needed
$ sudo rm -f /mnt/boot/uRamdisk
```

### 7.2.3 Boot it!
Now we unmount the partitions and insert the SD card into Raspberry Pi 4.
```bash
$ sudo umount /dev/sdb1
$ sudo umount /dev/sdb2
```
After powering up Raspberry Pi 4, we should get a `Busybox` shell if
successful. Unlike the `initramfs`-only case above, in this case, whatever
changes we make to the root filesystem will be persisted.

# Resources
[1] [Mastering Embedded Linux Programmin - Third Edition](https://www.amazon.com/Mastering-Embedded-Linux-Programming-potential/dp/1789530385)  
[2] [Linux From Scratch](https://www.linuxfromscratch.org/)  
[3] [How a toolchain is constructed](https://crosstool-ng.github.io/docs/toolchain-construction/)

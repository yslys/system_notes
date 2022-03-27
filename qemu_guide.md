## QEMU guide

[Source](https://www.youtube.com/watch?v=AAfFewePE7c&t=133s).

Recall that when we want to set up a VM either on VMWare or Virtual Box, what we need to do first is to download an ```.iso``` file that contains an OS. That ```.iso``` file is what we want to install onto an image. That image emulates a drive.

So now the question is - without those virtual machines with perfect GUI, how could we do that using QEMU?

Given an ```.iso``` file, we could do the following:
```
qemu-img create -f qcow2 Image.img 10G
```
+ ```create```: the create option to create something.
+ ```-f qcow2```: set the format to be (Qemu) Copy-on-Write with the second iteration (the latest one).
+ ```Image.img```: the name of the image with the ```.iso``` file installed onto.
+ ```10G```: the size of the image.

Now, for simplicity, we just emulate X86_64 architectures.

```
qemu-system-x86_64 -enable-kvm -cdrom OS_ISO.iso -boot menu=on \
-drive file=Image.img -m 2G
```
+ ```qemu-system-x86_64```: indicates we are emulating X86_64 architecture. Use other suffix for other architectures we want to emulate.
+ ```-enable-kvm```: start QEMU in KVM mode, with much better performance. This is the same as the argument ```accel=kvm``` of the ```-machine``` option. This is also equivalent to ```-accelkvm``` option.
+ ```-cdrom OS_ISO.iso```: we have to select our CD ROM so we can load this into our CD ROM slot. So this option selects an ```.iso``` to load as a CD.
+ ```-boot order=d```: specifies the boot order. ```order=d``` means load up the disk.
+ ```-boot menu=on```: turns on a menu you can use to select what you want to boot from.
+ ```-drive```: selects a virtual hard drive, ```file=Image.img``` means we are going to set the virtual hard drive to be a file - ```Image.img```.
+ ```-m 2G```: sets 2G of memory allocated for this virtual machine.

How could we get faster CPU emulation and faster graphical emulation? We can add several new options to the above command.
```
-cpu host
-smp 4
```
+ ```-cpu host```: set the CPU of the virtual machine to the same one on my computer.
+ ```-smp 4```: set the amount of cores to be 4.



#### Working with ISO and IMG files in QEMU
[Source](https://www.youtube.com/watch?v=JxSGT_3UU8w).

Given an ```.iso``` file which contains a distribution of Linux, what we can do to convert it into an image file?









## Embedded Linux from Scratch for RISC-V
Resources:
+ [YouTube: Embedded Linux from Scratch in 45 minutes, on RISC-V](https://www.youtube.com/watch?v=cIkTh3Xp3dA)
+ [GitHub: RISC-V Bringup](https://github.com/carlosedp/riscv-bringup/tree/master/Qemu)

### Table of Contents


### Preparation Phase (QEMU + Cross-Compiler)

#### Preparation for QEMU:
Run the following on Debian or Ubuntu distros to install QEMU: (qemu-install.sh)
```
# installation
sudo apt-get update
sudo apt-get install qemu-user-static qemu-system qemu-utils qemu-system-misc binfmt-support
```

After installing QEMU, we can execute the following to see the supported machines:
```
# verification
# The command below is the same as: qemu-system-riscv64 -machine help
qemu-system-riscv64 -M ?
```

The expected output should be:
```
Supported machines are:
microchip-icicle-kit Microchip PolarFire SoC Icicle Kit
none                 empty machine
sifive_e             RISC-V Board compatible with SiFive E SDK
sifive_u             RISC-V Board compatible with SiFive U SDK
spike                RISC-V Spike board (default)
virt                 RISC-V VirtIO board
```
With so many options, we are going to use the ```virt``` one, emulating VirtIO (virtual IO) peripherals, which is more efficient than emulating real hardware.

#### Build RISC-V Cross-Compiling Tool Chain
What is C Library?
+ The C library is an essential component of Linux
    + Interface between applications and kernel.
    + Provides standard C API to ease application development.
+ Several C libraries are available:
    + glibc: full featured, but rather big (2MB on ARM).
    + uClibc: better adapted to embedded use, smaller but supporting RISC-V 64.
    + musl: great for embedded use, more recent.
+ The choice of the C library must be made at cross-compiling toolchain generation time, since the GCC compiler is compiled against a specific C library.

Steps are as follows:
```
git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev libncurses-dev device-tree-compiler
./configure --prefix=/opt/riscv
sudo make linux -j6
export PATH=/opt/riscv/bin:$PATH
echo "export PATH=/opt/riscv/bin:$PATH" >> ~/.bashrc
```

Note: you could also build the riscv toolchain using buildroot. The usage is demonstrated in [YouTube: Embedded Linux from Scratch in 45 minutes, on RISC-V](https://www.youtube.com/watch?v=cIkTh3Xp3dA).

Testing: riscv-toolchain-verify.sh
```
# Generate a helloworld file
# Note that > means overwrite, >> means append to the end (with a newline at the beginning)
cat << EOF > helloworld.c
#include <stdio.h>
int main() {
    printf("hello world\n");
    return 0;
}
EOF

# Compile the file statically
riscv64-unknown-linux-gnu-gcc -static -o helloworld helloworld.c

# Execute the file with QEMU
qemu-riscv64-static helloworld

# clean the files
rm -rf helloworld helloworld.c
```
If you see ```hello world``` output, then it means your RISC-V cross-compiler is successfully installed.


### RISC-V privilege modes
RISC-V has three privilege modes (from low to high):
+ **U**ser (U-Mode): applications
+ **S**upervisor (S-Mode): OS kernel (U-Boot / Linux)
+ **M**achine (M-Mode): bootloader and firmware (OpenSBI / Uboot SPL)

Here are the typical combinations:
+ Simple embedded systems: **M**
+ Embedded systems with memory protection: **M**, **U**
+ Unix-style OS with virtual memory: **M**, **S**, **U**

Below is an illustration of three modes: (demonstrating the boot sequence, and decreasing privileges from bottom to top)
```
U mode        User space (Applications)
---------------------------------------------------
S mode    Operating System (U-Boot / Linux)
---------------------------------------------------
M mode     Firmware (OpenSBI / U-Boot SPL)
```

#### Why talking about RISC-V privilege modes?
You might be wondering why I discuss RISC-V privilege modes here. This is because the privilege mode defines the order during the boot process. What people would most likely to ignore is **What happens after we hit the boot button of the computer**. Since OS cannot run until being loaded to RAM, we need something else to direct the hardware to load the OS during the boot process. That is why there is BIOS, ROM, etc. The bootloader has the highest privilege level (M mode), and thus it is the bootloader that runs first after we hit the boot button.

Since we are now using QEMU which provides us a virtual hardware, we need to manually create the bootloader, the OS, etc. Therefore, I would follow the tutorial of the YouTube video and build the necessary elements during the boot process.
+ Linux (S Mode)
+ Boot loader: U-Boot (S Mode)
+ Firmware: OpenSBI (M Mode)


Note that in order to boot Linux kernel, we need to use U-Boot in S mode. U-Boot is required when building OpenSBI, the firmware. However, although we need to build U-Boot first and then the firmware, during the boot process, the firmware is required to start U-Boot.

#### Bootloader
We will start with U-Boot bootloader.

While compiling the bootloader (U-Boot), we need to explicitly specify the environment variable ```CROSS_COMPILE=``` to tell GCC that we are using the RISCV cross compiler, not the compiler on the host machine. This could be done in two ways:
```
# Note: On my host machine, after compiling the RISC-V cross compiler, the
# corresponding riscv64 gcc cross compiler is named: 
#         riscv64-unknown-linux-gnu-gcc
# Therefore, we need to eliminate gcc (the suffix), and do the following:

# 1st: set once, then not worrying about it at all.
export CROSS_COMPILE=riscv64-unknown-linux-gnu-

# 2nd: set it every time we cross-compile anything. For instance:
# Say we are building the Linux kernel for RISC-V architectures:
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j $(nproc)


# If the name of the executable of our cross-compiler is: riscv64-linux-gcc,
# then we need to do the following:
export CROSS_COMPILE=riscv64-unknown-linux-
```

We could also make a script to do that - riscv64-env.sh
```
cat << EOF > riscv64-env.sh
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
EOF

chmod 755 riscv64-env.sh
source riscv64-env.sh
```

**Build U-Boot:**

Next, we will start building U-Boot. The full script could be found in uboot-build.sh.

```
# Download U-Boot source
git clone https://github.com/U-Boot/U-Boot u-boot
cd u-boot
git checkout v2021.01

# Find U-Boot ready-made configurations for RISC-V
ls configs | grep riscv

# Choose the configuration for QEMU and U-Boot running in S Mode so that it can
# boot the Linux kernel. If we choose the configuration for M Mode, then we will
# not be able to start the Linux kernel
CROSS_COMPILE=riscv64-unknown-linux-gnu- make qemu-riscv64_smode_defconfig
CROSS_COMPILE=riscv64-unknown-linux-gnu- make -j$(nproc)
cd ..
```
Now, check the U-Boot directory, we could see a file uboot.bin located in:
```
u-boot/build/uboot.bin
```
This file will then be used to build OpenSBI, the firmware.


#### Build the firmware (M Mode): OpenSBI
What is OpenSBI (firmware, M Mode)?
+ Open Supervisor Binary Interface
+ A program that allows to switch from Machine Mode to Supervisor Mode. (U-Boot runs in Supervisor Mode.)
+ It is the firmware that is required / will be used to start U-Boot.

```
rm -rf opensbi
# download the source of OpenSBI
git clone https://github.com/riscv/opensbi.git
cd opensbi
git checkout v0.8

# compile OpenSBI (firmware)
# FW_PAYLOAD_PATH: the path to firmware payload (U-Boot)
make CROSS_COMPILE=riscv64-unknown-linux-gnu- \
     PLATFORM=generic \
     FW_PAYLOAD_PATH=../u-boot/u-boot.bin
cd ..
```
After that, we could find a binary file ```./opensbi/build/platform/generic/firmware/fw_payload.elf``` that contains U-Boot, which would be used by QEMU for boot process.

What is firmware payload?
+ I searched and found a [blog discussing Windows](https://blogs.windows.com/windowsdeveloper/2018/10/17/introducing-component-firmware-update/), in which it states that the payload of a package is a range of addresses and bytes to be programmed. (TODO: still not 100% understood yet.)


Note that, according to [OpenSBI README on GitHub](https://github.com/avpatel/opensbi#supported-sbi-version), OpenSBI v0.7 and up requires a Kernel version 5.7 or up.

The whole script could be found in opensbi-build.sh.

**Important:** Every time you update U-Boot, need to run opensbi-build.sh again, meaning that everytime we update U-Boot, need to re-build the firmware (OpenSBI).

#### First run with QEMU
It is excited that with the above components (U-Boot and OpenSBI), we are now able to start our virtual hardware, although we can do limited things with it for now since we have not yet built the kernel.

```
# -m: the amount of RAM in the emulated machine
# -smp: the number of CPUs in the emulated machine
# exit with Ctrl+A then X.
qemu-system-riscv64 -m 2G \
    -nographic \
    -machine virt \
    -smp 8 \
    -bios opensbi/build/platform/generic/firmware/fw_payload.elf
```

The expected output should be like the following, indicating U-Boot has started:
```
OpenSBI v0.8
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name       : riscv-virtio,qemu
Platform Features   : timer,mfdeleg
Platform HART Count : 8
Boot HART ID        : 0
Boot HART ISA       : rv64imafdcsu
BOOT HART Features  : pmp,scounteren,mcounteren,time
BOOT HART PMP Count : 16
Firmware Base       : 0x80000000
Firmware Size       : 152 KB
Runtime SBI Version : 0.2

MIDELEG : 0x0000000000000222
MEDELEG : 0x000000000000b109
PMP0    : 0x0000000080000000-0x000000008003ffff (A)
PMP1    : 0x0000000000000000-0xffffffffffffffff (A,R,W,X)


U-Boot 2021.01 (Mar 26 2022 - 16:32:30 -0700)

CPU:   rv64imafdcsu
Model: riscv-virtio,qemu
DRAM:  2 GiB
In:    uart@10000000
Out:   uart@10000000
Err:   uart@10000000
Net:   No ethernet found.
Hit any key to stop autoboot:  0 

Device 0: unknown device
scanning bus for devices...

Device 0: unknown device
No ethernet found.
No ethernet found.
=> 
```

#### Build Linux kernel
**Important Link:** https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git. This link allows us to download zipped Linux source tagged by Linus with a specific version without the need of git clone and checkout. Many thanks to Mark!



```
# download linux-5.11-rc3.tar.gz
# untar the zipped file
tar xf ...
cd linux-5.11-rc3/

# configure the kernel using default
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig

# Note that we could also set the environment variable manually by:
# create a file: riscv-env.sh, and add something to it:
#   export CROSS_COMPILE=riscv64-unknown-linux-gnu-
#   export ARCH=riscv
# save and execute: $ source riscv-env.sh

# compile the kernel
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j $(nproc)

# if you need to recompile the kernel over and over again, you would like it to
# compile faster, so you could execute the following:
CROSS_COMPILE=riscv64-unknown-linux-gnu- ARCH=riscv make -j $(nproc) CC="ccache riscv64-unknown-linux-gnu-gcc"
# We added a line: CC="ccache riscv64-unknown-linux-gnu-gcc" here.
# ccache: C cache, followed by the cross-compiler prefix.
```

After building the kernel, you would have the following files:
+ ```./vmlinux```: raw kernel in ELF format, not bootable, used for debugging
+ ```arch/riscv/boot/Image```: uncompressed bootable kernel
+ ```arch/riscv/boot/Image.gz```: compressed bootable kernel

Below are something that is worth knowing about:
+ ```CROSS_COMPILE=``` is the cross-compiler prefix - our cross-compiler is ```riscv64-linux-gcc```.
+ ```ARCH``` is the name of the sub-directory in arch/, corresponding to the target architecture.
+ ```$ make help | grep defconfig```: this command will grep all the configurations
+ ```CROSS_COMPILE=riscv64-unknown-linux-gnu- ARCH=riscv make -j $(nproc) CC="ccache riscv64-unknown-linux-gnu-gcc"``` allows to build kernel faster when building kernel for many times.

#### Second run with QEMU (directly with OpenSBI and Linux kernel, without U-Boot)
Now that we have the kernel, you would likely to expect what will happen if we boot QEMU with the kernel. But before that, we need to do some preparations.

The preparation that we need to do is to re-build OpenSBI, passing the newly built kernel to it via ```FW_PAYLOAD=```, so that the payload contains the Linux image, instead of U-Boot.

```
# rebuild OpenSBI against the Linux kernel
cd opensbi
make PLATFORM=generic FW_PAYLOAD_PATH=../linux-5.11-rc3/arch/riscv/boot/Image
cd ..

# -append "console=ttyS0": this means we are passing the kernel command line to Linux boot

qemu-system-riscv64 -m 2G \
    -nographic \
    -machine virt \
    -smp 8 \
    -bios opensbi/build/platform/generic/firmware/fw_payload.elf \
    -append "console=ttyS0"
```

However, this approach eliminates the U-Boot, which is not similar to the boot process in real world. What we want is a process that involves OpenSBI, U-Boot and Linux kernel. In other words, the first-stage bootloader (OpenSBI) starts the bootloader (U-Boot), then bootloader starts the Kernel, and then starts user space.

Therefore, we need to set the U-Boot environment to load the Linux kernel and to specify the Linux kernel command line, i.e. instead of letting OpenSBI invoke the Linux kernel, we let OpenSBI invoke U-Boot, then U-Boot invokes Linux kernel. For this purpose, we need some storage space to store the U-Boot environment, to load the kernel binary, and the storage space also contains the file system that Linux will boot on.

### Disk Image
First create a 4GB disk image using ```dd```command.
```
# create a disk image using dd command
#   if=/dev/zero: the input file is the zero device
#   of=disk.img: this is the output file, i.e. the disk image we want
#   bs=4K: the block size is 4KB
#   count=: the number of disk blocks is 1048576 (2^20).
dd if=/dev/zero of=disk.img bs=4K count=1048576
```
Then we will use ```cfdisk``` command to create two partitions in this image
+ The first 64MB primary partition (type W95 FAT32 (LBA)), marked as bootable
+ The second partition with remaining space (default type: Linux)

```
cfdisk disk.img

# After the above command, we will see a prompt saying "Select label type" 
# since the device (disk image we just created) does not contain a recognized 
# partition table.
#       Select "dos" as the label type.
# Now, we are going to create a new partition:
#       Select "New"
#       Set "Partition size" to be 64M
#       Select "Primary"
# After creating the new partition, we could see that the disk now contains two 
# partitions. Wee need to set the first partition to be bootable.
#       Make sure the cursor is selecting the first partition
#       Select "Bootable" - you could see an asterisk(*) on the Boot column.
# Next, we need to set the partition type of the first partition to be 
#       Select "Type"
#       Scroll up and select "c W95 FAT32 (LBA)" (c is the index)

# Now we have finished configuring the first partition, we could then start
# configuring the second one.
#       Move the cursor to "Free space" (second partition)
#       Select "New"
#       No need to change the size in this case
#       Select "Primary"
# We can see that the type of the second partition is Linux by default.

# Now that we are done creating two partitions, we could write the partition
# table to the disk.
#       Select "Write"
#       Say yes
#       Select "Quit"
```

After partitioned the disk image, we could check it using ```fdisk```:
```
$ sudo fdisk -lu disk.img
(The output should be...)
Disk disk.img: 4 GiB, 4294967296 bytes, 8388608 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x067927db

Device     Boot  Start     End Sectors Size Id Type
disk.img1  *      2048  133119  131072  64M  c W95 FAT32 (LBA)
disk.img2       133120 8388607 8255488   4G 83 Linux
```

We can also try the following command provided by QEMU:
```
qemu-img info disk.img
```

After that, we could access the partitions in the disk image using ```losetup``` command (lo-setup):
```
# --partscan: partition scan, allows to automatically detect the partitions
# inside a disk image rather than having to manually define the offset and
# things like that, and such flag will setup loop devices for the partitions
# inside the disk image. In other words, it is convenient.
sudo losetup -f --show --partscan disk.img
```
On my host machine, the output of ```losetup``` is:
```
/dev/loop10
```
```losetup``` command associates loop devices (in ```/dev/loop*```) with regular files or block devices (our disk.img in this case). In other words, since the output is ```/dev/loop10```, it means the two partitions have been associated with ```/dev/loop10p1``` and ```/dev/loop10p2``` respectively. In my opinion, it is similar to mount command.


How to access the two disk partitions?
+ The two disk partitions could be found in ```/dev``` directory.
+ Use ```ls -al /dev/loop10*```, and we can see ```/dev/loop10p1``` and ```/dev/loop10p2```.


Now that we can access the two disk partitions, we could then format those partitions:
```
# For the 1st partition (/dev/loop10p1):
# Recall that it has type W95 FAT32 (LBA).

# mkfs.vfat is same as mkdosfs command. It is used to create an MS-DOS file 
# system under Linux on a device (usually a disk partition).
#   -f num-of-FATs: specify the number of File Allocation Tables in the file 
#                   system. Default: 2. Currently, Linux MS-DOS file system
#                   does not support more than 2 FATs. 
#   -F FAT-size: specify the type of File Allocation Tables used, 12, 16 or 32
#                bit. If nothing is specified, will automatically select
#                between 12, 16 and 32 bit, whatever fits better for the file
#                system size.
#   -n volume-name: sets the volume name (label) of the file system. The volume 
#                   name can be up to 11 characters long. The default is no
#                   label.
# For more info: https://linux.die.net/man/8/mkfs.vfat
sudo mkfs.vfat -F 32 -n boot /dev/loop10p1

# For the 2nd partition (/dev/loop10p2):
# Recall that it has type Linux.
# mkfs.ext4 creates an EXT4 file system usually in a disk partition.
#   -L new-volume-label: set the volume label for the filesystem to
#                        new-volume-label. The maximum length of the volume
#                        label is 16 bytes.
sudo mkfs.ext4 -L rootfs /dev/loop10p2
```
















Now we are in a concole-like space where we could type in some commands we like.

## Resources:
https://www.youtube.com/watch?v=Y-FUvi1z1aU
https://www.youtube.com/watch?v=cIkTh3Xp3dA&t=885s
https://www.youtube.com/watch?v=iWQRV-KJ7tQ
https://www.youtube.com/watch?v=DI6DSVq0J6s
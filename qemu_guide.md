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


Note that in order to boot Linux kernel, we need to use U-Boot in S mode. U-Boot is required when building OpenSBI, the firmware. However, although we need to build U-Boot first and then the firmware (OpenSBI), during the boot process, the firmware(OpenSBI) is required to start U-Boot.

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
# git checkout v0.8
# Do not use v0.8, as this would lead an error while "saveenv" in U-Boot mode.
# In detail, saveenv will not flush to virtio device (our disk), so after
# reboot, the environment variable will remain unchanged even it displayed:
#   "Saving Environment to FAT... OK" after we saveenv.

# compile OpenSBI (firmware)
# FW_PAYLOAD_PATH: the path to firmware payload (U-Boot)
# PLATFORM=generic: under opensbi root directory, it has a directory called
#                   platform, "generic" means we are using 
                    ./opensbi/platform/generic to build openSBI
make CROSS_COMPILE=riscv64-unknown-linux-gnu- PLATFORM=generic FW_PAYLOAD_PATH=../u-boot/u-boot.bin
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
make CROSS_COMPILE=riscv64-unknown-linux-gnu- \
    PLATFORM=generic \
    FW_PAYLOAD_PATH=../linux-5.11-rc3/arch/riscv/boot/Image
cd ..

# -append "console=ttyS0": this means we are passing the kernel command line to
# Linux boot

qemu-system-riscv64 -m 2G \
    -nographic \
    -machine virt \
    -smp 8 \
    -bios opensbi/build/platform/generic/firmware/fw_payload.elf \
    -append "console=ttyS0"
```

However, this approach eliminates the U-Boot, which is not similar to the boot process in real world. What we want is a process that involves OpenSBI, U-Boot and Linux kernel. In other words, the first-stage bootloader (OpenSBI) starts the bootloader (U-Boot), then bootloader starts the Kernel, and then starts user space.

Therefore, we need to set the U-Boot environment to load the Linux kernel and to specify the Linux kernel command line, i.e. instead of letting OpenSBI invoke the Linux kernel, we let OpenSBI invoke U-Boot, then U-Boot invokes Linux kernel. For this purpose, we need some storage space to store the U-Boot environment, to load the kernel binary, and the storage space also contains the file system that Linux will boot on. Such storage space is what we called "disk image".

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

What we are doing next is copy our kernel image to our first disk partition, so that U-Boot will be able to figure it out and boot Linux from there.
```
# create a directory under /mnt, to be mounted by /dev/loop10p1 (first disk
# partition)
sudo mkdir /mnt/boot
sudo mount /dev/loop10p1 /mnt/boot

# copy the Linux image to our first disk partition
cp linux-5.11-rc3/arch/riscv/boot/Image /mnt/boot/
# Permission denied! Need to append sudo. But can we make it faster?
#   By using !! to represent the last executed command
sudo !!

sudo umount /mnt/boot
sudo rm -rf /mnt/boot
```

Now that we have our Linux image in the first partition, we need to tell U-Boot about it. In other words, for U-Boot, it does not know it needs find the environment in a FAT partition (the first partition) on a virtio disk yet, and we need to tell it, by adding support for finding the FAT environment. But how? By reconfigure and then recompile it.

```
cd u-boot

# reconfigure it
CROSS_COMPILE=riscv64-unknown-linux-gnu- make menuconfig
# Below are what we want to configure:
# CONFIG_ENV_IS_IN_FAT=y, specifying the environment will be in FAT partition
# CONFIG_ENV_FAT_INTERFACE="virtio", the FAT will be in a virtIO device
# CONFIG_ENV_FAT_DEVICE_AND_PART="0:1", use the first partition from the first
#       virtIO device. This might be a bit confusing. It has the format of
#       "index_of_virtIO_device:index_of_partition_within_the_device".
#       index_of_virtIO_device starts from 0;
#       index_of_partition_within_the_device starts from 1; (/dev/loop10p1)
# If we only modify u-boot/.config:
#       - add CONFIG_ENV_IS_IN_FAT=y
#       - add CONFIG_ENV_FAT_INTERFACE="virtio"
#       - add CONFIG_ENV_FAT_DEVICE_AND_PART="0:1"
#       - add CONFIG_ENV_FAT_FILE="uboot.env"
#       - For MMC device number and partition number, can set default to be 0:
#           - CONFIG_SYS_MMC_ENV_DEV=0
#           - CONFIG_SYS_MMC_ENV_PART=0

# To use the menuconfig interface, do the following:
#       Select "Environment"
#       Select "Environment is in a FAT filesystem"
#       Then there will be some new options appeared
#       Select "Name of the block device for the environment", hit enter
#       Type in: "virtio" (excluding double quotes), then "ok"
#       Select "Device and partition for where to store the environemt in FAT"
#       Hit enter, type in: "0:1" (excluding double quotes), then "ok" 
#       
# Note: this approach will generate the FAT file to use for the environment, 
#       with the filename being uboot.env.
# For more details, check the next paragraph.


# recompile it
CROSS_COMPILE=riscv64-unknown-linux-gnu- make -j$(nproc)
```

#### How .config works?
What I would be discussing below is how the configuration works:

I did not know how to do it at first, but I solved it later. The first thing I did was to execute ```make menuconfig```. There are several options but I did not know which option to choose. Since we want to configure three things:
+ ```CONFIG_ENV_IS_IN_FAT=y```
+ ```CONFIG_ENV_INTERFACE="virtio"```
+ ```CONFIG_ENV_FAT_DEVICE_AND_PART="0:1"``` 
They all have prefix of ```CONFIG_ENV```, meaning "environment". And there is actually "Environment" option in menuconfig. So I selected "Environment", and then found the option "Environment is in a FAT filesystem". This corresponds to ```CONFIG_ENV_IS_IN_FAT```. Therefore, I selected that option, and saved, and in the newly generated .config file, it shows the difference. From ```# CONFIG_ENV_IS_IN_FAT is not set``` to ```CONFIG_ENV_IS_IN_FAT=y```. I later noticed that, since we have been focusing on environment configurations, there is also a directory ```./u-boot/env/```, in which it contains a file called ```Kconfig```, which contains the following:
```
config ENV_IS_IN_FAT
	bool "Environment is in a FAT filesystem"
	depends on !CHAIN_OF_TRUST
	default y if ARCH_BCM283X
	default y if ARCH_SUNXI && MMC
	default y if MMC_OMAP_HS && TI_COMMON_CMD_OPTIONS
	select FS_FAT
	select FAT_WRITE
	help
	  Define this if you want to use the FAT file system for the environment.
```
If we grep ```ENV_IS_IN_FAT``` under ```u-boot/env/``` directory by ```$ grep -r ENV_IS_IN_FAT .```, we can see that:
```
./env.c:#ifdef CONFIG_ENV_IS_IN_FAT
```
which means, the ```.config``` file is responsible for defining or not defining the macro ```ENV_IS_IN_FAT``` in ```u-boot/env/env.c``` file. Hence, if in ```.config``` file, ```CONFIG_ENV_IS_IN_FAT=y```, then the code between ```#ifdef CONFIG_ENV_IS_IN_FAT``` and ```#endif``` in ```u-boot/env/env.c``` file will be executed.

Remember that we mentioned before that "everytime we update U-Boot, need to re-build the firmware (OpenSBI)". Since we have re-compiled U-Boot, we need to re-compile OpenSBI as well
```
cd opensbi
make clean
make CROSS_COMPILE=riscv64-unknown-linux-gnu- PLATFORM=generic FW_PAYLOAD_PATH=../u-boot/u-boot.bin
```

#### Third run with QEMU (with an empty rootfs)
After re-building the firmware (OpenSBI), we can now run QEMU again!
```
qemu-system-riscv64 -m 2G -nographic -machine virt -smp 8 \
    -bios opensbi/build/platform/generic/firmware/fw_payload.elf \
    -drive file=disk.img,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0
```
Remember to hit enter immediately after the above command is executed, i.e. it would display:
```
Hit any key to stop autoboot:  3
Hit any key to stop autoboot:  2
Hit any key to stop autoboot:  1
```
So hit any key in-between, to stay in U-Boot. we would see the following output:

```
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


U-Boot 2021.01 (Mar 27 2022 - 00:51:45 -0700)

CPU:   rv64imafdcsu
Model: riscv-virtio,qemu
DRAM:  2 GiB
In:    uart@10000000
Out:   uart@10000000
Err:   uart@10000000
Net:   No ethernet found.
Hit any key to stop autoboot:  0 
=>
```

Now, if we execute ```fatls virtio 0:1```, the output would be:
```
=> fatls virtio 0:1
 19804672   Image

1 file(s), 0 dir(s)
```
Only one file called Image.


What we need to do next is to save an environment variable using ```setenv```:
```
# set an environment variable foo to be bar
setenv foo bar

# save an environment
saveenv     # output should be: "Saving Environment to FAT... OK"

# print the environment variable foo
printenv foo # the expected output should be: foo=bar.

# fatls again:
fatls virtio 0:1
```
The output should now contains two files:
```
=> fatls virtio 0:1
 19804672   Image
   131072   uboot.env

2 file(s), 0 dir(s)
```
Now, if we restart QEMU, and still stay in U-Boot mode, execute ```fatls virtio 0:1```, the output should be the same as above.

**So, as long as we have the uboot.env file shown, and we are sure that the changes we made to env are persistently saved to virtio device (after reboot), everything would be good.**

Note that the commands we were executing belong to U-Boot commands. For a full list and description of all commands supported by U-Boot, refer to U-Boot reference manual: https://hub.digi.com/dp/path=/support/asset/u-boot-reference-manual/.

#### Start Linux from U-Boot
Now, we should do something to start Linux from U-Boot:

Requirements for booting Linux:
+ Image: either Linux kernel image or the Initramfs image.
+ Device Tree Binary (DTB).
+ Set the Linux arguments (kernel command line)

**1. Image**:
To boot the Linux kernel, U-Boot needs to load a Linux kernel image. In our case, we are using virtIO device, so we load the kernel image from virtio disk to RAM. We can find a suitable RAM address by using ```bdinfo``` command in U-Boot. Such command will print board info structure, including the start and end of RAM address.

Hence, in order to tell U-Boot to do it, we need to execute the following (which we will not execute, instead, we would save it to the environment variable so that we do not need to execute it everytime we boot):
```
# fatload: load binary file from a dos filesystem
# load binary file @Image in the first(@0) @virtio dos filesystem's first (@1) 
# partition into RAM, with target address being 84000000
fatload virtio 0:1 84000000 Image
```
Note that the Image can also be the image of an Initramfs, which is a file system in RAM that Linux can use, but we will not use Initramfs in this demo.


**2. Device Tree Binary (DTB)**:
A binary that lets kernel know which SoC and devices we have. This allows the same kernel to support many different SoCs and boards at the same time.
+ Normally, DTB files are compiled from DTS files in ```arch/riscv/boot/dts/```.
+ However, there is no such DTS file for the RISC-V QEMU virt board.
+ According to [mailing list](https://tinyurl.com/y4ae5ptd), the DTB is built by QEMU and passed by QEMU to OpenSBI, and then to U-Boot. So actually we could access the loaded DTB at address ```${fdtcontroladdr}``` in RAM at U-Boot stage.

**3. Linux kernel command line (Image + DTB)**:
We need to set the Linux arguments (kernel command line):
```
# setenv: set an environment variable @bootargs to 'root=...'
# root=/dev/vda2 : tell Linux to mount the root file system from the second
#                  partition of the virtual disk (vda2).
# rootwait : wait for the root device to be ready before trying to mount it.
# console= : set the device to send Linux booting messages to.
# console=ttyS0 : directs the Linux kernel booting messages to the first serial
#                 line (ttyS0). If you put the incorrect value here, you may
#                 see no message at all.
# earlycon=sbi : allows to have more booting messages even before the console
#                driver is initialized. earlycon stands for Early Console.
setenv bootargs 'root=/dev/vda2 rootwait console=ttyS0 earlycon=sbi'
```


To boot the Linux Image file, we need this command (which we will NOT execute, instead, we would save it to environment variable) that listed below for reference:
```
# booti <Linux address> <Initramfs address> <DTB address>
booti 0x84000000 - ${fdtcontroladdr}
```

So far we have been discussing about two commands: (1) load Image to a specific address in RAM, and (2) boot Linux Image file and DTB. What we need to do next is to combine them together and save them into the environment variable. Hence, we will define the default series of commands (in this case, 2) that U-Boot will automatically run after a configurable delay (```bootdelay``` environment variable):
```
setenv bootcmd 'fatload virtio 0:1 84000000 Image; booti 0x84000000 - ${fdtcontroladdr}'

# then save
saveenv
```


**4. To Conclude:**
If you want to manually set environment variable everytime you boot, then remember to first press any key to stay in U-Boot mode, then:
```
setenv bootargs 'root=/dev/vda2 rootwait console=ttyS0 earlycon=sbi'
fatload virtio 0:1 84000000 Image
booti 0x84000000 - ${fdtcontroladdr}
```

If you want to save the above commands to environment variable persistently so that no need to execute those everytime we boot into U-Boot state, you could do the following, which is what I would recommend:
```
setenv bootargs 'root=/dev/vda2 rootwait console=ttyS0 earlycon=sbi rw'
setenv bootcmd 'fatload virtio 0:1 84000000 Image; booti 0x84000000 - ${fdtcontroladdr}'
saveenv

# then can execute $ boot to boot.
```

In any case, we would result in the following output:
```
...
[    0.463255] VFS: Mounted root (ext4 filesystem) readonly on device 254:2.
[    0.465447] devtmpfs: error mounting -2
[    0.485072] Freeing unused kernel memory: 2144K
[    0.485971] Run /sbin/init as init process
[    0.486836] Run /etc/init as init process
[    0.487150] Run /bin/init as init process
[    0.487442] Run /bin/sh as init process
[    0.487960] Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.
[    0.488556] CPU: 4 PID: 1 Comm: swapper/0 Not tainted 5.11.0-rc3 #1
[    0.488869] Call Trace:
[    0.489014] [<ffffffe0000047b8>] walk_stackframe+0x0/0xaa
[    0.489241] [<ffffffe0006d3a7e>] show_stack+0x32/0x3e
[    0.489408] [<ffffffe0006d644c>] dump_stack+0x72/0x8c
[    0.489562] [<ffffffe0006d3c92>] panic+0x100/0x2b6
[    0.489763] [<ffffffe0006dda82>] kernel_init+0xec/0xf8
[    0.489954] [<ffffffe0000032f2>] ret_from_exception+0x0/0xc
[    0.490322] SMP: stopping secondary CPUs
[    0.491857] ---[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance. ]---
```
This is the expected output since it mounts the root file system as instructed, but it fails to execute an init process according to the output line ```Kernel panic - not syncing: No working init found.```. That is fine since we have not yet put anything in our root file system (the second partition) yet.



#### Build our Root File System
There are two options: BusyBox, or building Ubuntu rootfs on our own.

##### BusyBox:
```
wget https://busybox.net/downloads/busybox-1.33.0.tar.bz2
tar xf busybox-1.33.0.tar.bz2
cd busybox-1.33.0/
// TODO:
```


##### Building our own Ubuntu rootfs

BusyBox version is fast and easier, but the problem is that it lacks a lot of useful binaries like git, etc. Since we want to do ```apt-get update```, and make the system a Ubuntu-like system, I choose the second approach, which is described in this [link](https://github.com/carlosedp/riscv-bringup/blob/master/Ubuntu-Rootfs-Guide.md). This link describes the similar approach as the YouTube video, but I ended up with errors that I did not know how to solve. But the link still contains some useful resources that we could borrow (like the kernel modules part).

Note that the line ```echo "root:riscv" | chpasswd``` sets the login to be root, and passward to be riscv.

The self-built Ubuntu rootfs could be found in ```Ubuntu-Hippo-rootfs.tar.gz``` file.

In case we would not be able to access the link in the future, I would paste the commands below:
> This guide walks thru the build of a Ubuntu root filesystem from scratch. Ubuntu supports riscv64 packages from Focal, released on 04/2020.

> Here we will build an Ubuntu Hirsute Hippo (latest version). I recommend doing this process can be done on a recent Ubuntu host (Focal or newer).

```
# Install pre-reqs
sudo apt install debootstrap qemu qemu-user-static binfmt-support dpkg-cross --no-install-recommends

# Generate minimal bootstrap rootfs
sudo debootstrap --arch=riscv64 --foreign hirsute ./temp-rootfs http://ports.ubuntu.com/ubuntu-ports

# chroot to it and finish debootstrap
sudo chroot temp-rootfs /bin/bash

/debootstrap/debootstrap --second-stage

# Add package sources (two options: 20.04 Focal, and 21.04 Hirsute Hippo)
# Pick the one that you want

# For Focal Fossa (Ubuntu 20.04) (note, when generating minimal bootstrap rootfs
# --foreign needs to be changed as well)
cat >/etc/apt/sources.list <<EOF
deb http://archive.ubuntu.com/ubuntu/ focal main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ focal main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ focal-updates main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ focal-security main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ focal-backports main restricted universe multiverse

deb http://archive.canonical.com/ubuntu focal partner
deb-src http://archive.canonical.com/ubuntu focal partner
EOF

# For Hirsute Hippo (Ubuntu 21.04)
cat >/etc/apt/sources.list <<EOF
deb http://ports.ubuntu.com/ubuntu-ports hirsute main restricted

deb http://ports.ubuntu.com/ubuntu-ports hirsute-updates main restricted

deb http://ports.ubuntu.com/ubuntu-ports hirsute universe
deb http://ports.ubuntu.com/ubuntu-ports hirsute-updates universe

deb http://ports.ubuntu.com/ubuntu-ports hirsute multiverse
deb http://ports.ubuntu.com/ubuntu-ports hirsute-updates multiverse

deb http://ports.ubuntu.com/ubuntu-ports hirsute-backports main restricted universe multiverse

deb http://ports.ubuntu.com/ubuntu-ports hirsute-security main restricted
deb http://ports.ubuntu.com/ubuntu-ports hirsute-security universe
deb http://ports.ubuntu.com/ubuntu-ports hirsute-security multiverse
EOF



# Install essential packages
# Need to install **net-tools** for network interfaces (e.g. ifconfig)
apt-get update
apt-get upgrade

# perl: warning: Setting locale failed.
# perl: warning: Please check that your locale settings:
#     LANGUAGE = (unset),
#     LC_ALL = (unset),
#     LC_CTYPE = "en_US.UTF-8",
#     LANG = "en_US.UTF-8"
#     are supported and installed on your system.
# Solution:
update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

# Install some packages
apt-get install --no-install-recommends -y util-linux haveged openssh-server \
systemd kmod initramfs-tools conntrack ebtables ethtool iproute2 iptables \
mount socat ifupdown iputils-ping vim dhcpcd5 neofetch sudo chrony net-tools \
make git \


# Install packages for RISC-V cross compiler
apt-get install autoconf automake autotools-dev curl python3 libmpc-dev \
libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool \
patchutils bc zlib1g-dev libexpat-dev

# Install packages for compiling Linux kernel
apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison


# Create base config files
mkdir -p /etc/network
cat >>/etc/network/interfaces <<EOF
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF

cat >/etc/resolv.conf <<EOF
nameserver 1.1.1.1
nameserver 8.8.8.8
EOF

cat >/etc/fstab <<EOF
LABEL=rootfs	/	ext4	user_xattr,errors=remount-ro	0	1
EOF

echo "Ubuntu-riscv64" > /etc/hostname

# Disable some services on Qemu
ln -s /dev/null /etc/systemd/network/99-default.link
ln -sf /dev/null /etc/systemd/system/serial-getty@hvc0.service

# Set root passwd
echo "root:riscv" | chpasswd

sed -i "s/#PermitRootLogin.*/PermitRootLogin yes/g" /etc/ssh/sshd_config

# Clean APT cache and debootstrap dirs
rm -rf /var/cache/apt/

# Exit chroot
exit
sudo tar -cSf Ubuntu-Hippo-rootfs.tar -C temp-rootfs .
gzip Ubuntu-Hippo-rootfs.tar
rm -rf temp-rootfs
```


So now the Ubuntu Hippo rootfs could be found in ```Ubuntu-Hippo-rootfs.tar.gz``` file. What we need to do next is to generate the kernel modules of Linux that we are using.


Here is the third approach by directly downloading the pre-built Ubuntu Focal tarball:
```
# Download the tar file
wget -O rootfs.tar.bz2 https://github.com/carlosedp/riscv-bringup/releases/download/v1.0/UbuntuFocal-riscv64-rootfs.tar.gz

# Extract the file into rootfs directory
mkdir rootfs
tar xvf rootfs.tar.bz2 -C rootfs

# Change root so that we can install net-tools package (ifconfig)
sudo chroot rootfs/ /bin/bash

# Try apt-get update
apt-get update # W: An error occurred during the signature verification. 

# Solution: this is because of the file permission of /tmp directory
ls -lad /tmp # this should allow you to see the permission (should be 1777)
# but it showed 1000. So we need to change it to 1777
chmod 1777 /tmp

# Then run apt-get update again (it might be too slow, so update /etc/apt/sources.list)
apt-get update
apt-get upgrade
# Install some packages
apt-get install --no-install-recommends -y util-linux haveged openssh-server \
systemd kmod initramfs-tools conntrack ebtables ethtool iproute2 iptables \
mount socat ifupdown iputils-ping vim dhcpcd5 neofetch sudo chrony net-tools \
make git wget

# Install packages for compiling the kernel
apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison

# Compile the kernel under /usr/src/ directory



```


##### Generating Kernel Modules
generate-kernel-modules.sh:
```
#!/bin/sh

# remember to change this if you are using another version of Linux
version="5.11-rc3"  # version=`cat include/config/kernel.release`
pathtolinux="./linux-5.11-rc3"

cd $pathtolinux
rm -rf modules_install
mkdir -p modules_install
CROSS_COMPILE=riscv64-unknown-linux-gnu- ARCH=riscv make modules_install INSTALL_MOD_PATH=./modules_install

cd ./modules_install/lib/modules

tar -cf kernel-modules-$version.tar .
gzip kernel-modules-$version.tar

mv ./kernel-modules-$version.tar.gz ../../../
echo "generating kernel modules finished"
echo "kernel modules: $pathtolinux/kernel-modules-$version.tar.gz"
```

In our case, the generated kernel module zip file is: ```./linux-5.11-rc3/kernel-modules-5.11-rc3.tar.gz```.

#### Copy rootfs and kernel modules to second partition of our disk image
```
#!/bin/sh

# remember to change this if you are using another version of Linux
version="5.11-rc3"  # version=`cat include/config/kernel.release`
pathtolinux="./linux-5.11-rc3"

# Mount the second partition of disk image to /mnt/rootfs
sudo mkdir -p /mnt/rootfs
sudo mount /dev/loop10p2 /mnt/rootfs

# Check to see what is in rootfs of disk image
# In my case, it has only one directory: lost+found
ls /mnt/rootfs 

# Copy rootfs we generated to disk image (can ls again to see the change)
sudo tar xvf Ubuntu-Hippo-rootfs.tar.gz -C /mnt/rootfs

# Copy Kernel modules to disk image
sudo mkdir -p /mnt/rootfs/lib/modules
sudo tar xvf $pathtolinux/kernel-modules-$version.tar.gz -C /mnt/rootfs/lib/modules

# Copy kernel Image file
sudo mkdir -p /mnt/rootfs/boot/extlinux
sudo cp $pathtolinux/arch/riscv/boot/Image /mnt/rootfs/boot/mvlinuz-$version

cat << EOF | sudo tee /mnt/rootfs/boot/extlinux/extlinux.conf
menu title RISC-V Qemu Boot Options
timeout 100
default kernel-$version

label kernel-$version
        menu label Linux kernel-$version
        kernel /boot/vmlinuz-$version
        initrd /boot/initrd.img-$version
        append earlyprintk rw root=/dev/vda1 rootwait rootfstype=ext4 LANG=en_US.UTF-8 console=ttyS0

label rescue-kernel-$version
        menu label Linux kernel-$version (recovery mode)
        kernel /boot/vmlinuz-$version
        initrd /boot/initrd.img-$version
        append earlyprintk rw root=/dev/vda1 rootwait rootfstype=ext4 LANG=en_US.UTF-8 console=ttyS0 single
EOF

```


```
sudo qemu-system-riscv64 -m 2G -nographic -machine virt -smp 8 \
    -bios opensbi/build/platform/generic/firmware/fw_payload.elf \
    -drive file=disk.img,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -netdev tap,id=tapnet,ifname=tap2,script=no,downscript=no \
    -device virtio-net-device,netdev=tapnet


qemu-system-riscv64 -nographic -machine virt -m 4G \
 -bios opensbi/build/platform/generic/firmware/fw_payload.elf \
 -drive file=disk.img,format=raw,id=hd0 \
 -device virtio-blk-device,drive=hd0 \
 -device virtio-net-device,netdev=usernet -netdev user,id=usernet,hostfwd=tcp::22222-:22

```
# Now we can detach the file or device (disk.img) with the loop device (/dev/loop10)
```
sudo losetup -d /dev/loop10
```

## Resources:
https://www.youtube.com/watch?v=Y-FUvi1z1aU
https://www.youtube.com/watch?v=cIkTh3Xp3dA&t=885s
https://www.youtube.com/watch?v=iWQRV-KJ7tQ
https://www.youtube.com/watch?v=DI6DSVq0J6s
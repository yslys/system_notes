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




## Resources:
https://www.youtube.com/watch?v=Y-FUvi1z1aU
https://www.youtube.com/watch?v=cIkTh3Xp3dA&t=885s
https://www.youtube.com/watch?v=iWQRV-KJ7tQ
https://www.youtube.com/watch?v=DI6DSVq0J6s
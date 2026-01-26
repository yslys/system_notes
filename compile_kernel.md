## Compiling a Linux kernel

First, clone the kernel source code with the version you like. Then install the dependencies
```
sudo apt update
sudo apt install -y build-essential bc bison flex libssl-dev libelf-dev dwarves pkg-config git rsync wget curl libncurses-dev
```

Then, we need to configure the kernel before building it:
```
cp /boot/config-$(uname -r) .config
make olddefconfig

scripts/config --enable CONFIG_DAMON \
               --enable CONFIG_DAMON_VADDR \
               --enable CONFIG_DAMON_PADDR \
               --enable CONFIG_DAMON_SYSFS \
               --enable CONFIG_DAMON_RECLAIM \
               --enable CONFIG_DAMON_LRU_SORT \
               --enable CONFIG_TRANSPARENT_HUGEPAGE

# Below is required to resolve dependencies
make olddefconfig
```

To verify the config file is written correctly:
```
grep -E "CONFIG_DAMON|CONFIG_TRANSPARENT_HUGEPAGE" .config
```

Compile the kernel:
```
make -j$(nproc)
```

Install kernel modules and kernel itself:
```
sudo make modules_install
sudo make install
```

Update Initramfs:
```
sudo update-initramfs -c -k $(make kernelrelease)
```

Verify by:
```
ls /boot/initrd.img-$(make kernelrelease)
```
Update grub, then reboot:
```
sudo update-grub
sudo reboot
```




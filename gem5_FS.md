It's been a while since the last time I set up the full system mode of gem5 with Linux kernel. Recently I upgraded my desktop and finally gave it a chance to dual boot Ubuntu on a system installed with Windows 11. It worked but it means I will have to set up the system again.

So I installed RISC-V cross compiler following this [link](https://github.com/riscv-collab/riscv-gnu-toolchain), which was what I followed a year ago, hoping to use such tool chain to cross compile the Linux kernel (this [link](https://risc-v-getting-started-guide.readthedocs.io/en/latest/linux-qemu.html)); however, an error occurred:
```
csrs 0x100,s4', extension `zicsr' required
```

### First try (did not work)
I found the corresponding issue [here](https://github.com/riscv-collab/riscv-gnu-toolchain/issues/1280), and the solution I've tried so far is to pass this flag to the configure script: `--with-arch=rv64gc`
```
./configure --prefix=/opt/riscv --with-arch=rv64gc
sudo make linux
```

### Second try (did not work)
```
./configure --prefix=/opt/riscv --with-arch=rv64gc_zicsr
sudo make linux
```

This link is useful: https://groups.google.com/a/groups.riscv.org/g/sw-dev/c/aE1ZeHHCYf4

### Third try (working solution)
First of all, we should build the riscv cross compiler following the instructions shown on the Github repo, i.e.
```
./configure --prefix=/opt/riscv
make linux
```
Then, when we try to build the Linux kernel, we should modify the Makefile. How to modify is based on [this issue](https://github.com/OpenXiangShan/XiangShan/issues/2545).

In `linux.../arch/riscv/Makefile`:
```
# Search for "KBUILD_AFLAGS += -march=", then append "_zicsr_zifencei"
# Search for "KBUILD_CFLAGS += -march=", then append "_zicsr_zifencei"
# In my case, I modified it to be the following:
KBUILD_CFLAGS += -march=$(subst fd,,$(riscv-march-y))_zicsr_zifencei
KBUILD_AFLAGS += -march=$(riscv-march-y)_zicsr_zifencei
```
Then we are good to go.

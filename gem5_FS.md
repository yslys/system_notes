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

### Second try
```
./configure --prefix=/opt/riscv --with-arch=rv64gc_zicsr
sudo make linux
```

This link is useful: https://groups.google.com/a/groups.riscv.org/g/sw-dev/c/aE1ZeHHCYf4

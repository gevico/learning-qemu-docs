目前支持 RISCV 虚拟化扩展的硬件不是很多，但我们可以使用 QEMU 来模拟这个功能。

基本思路是采用 QEMU 软件模拟一个 RISCV SoC（virt Machine），在上面运行 Ubuntu 发行版，然后在 Ubuntu 上使用虚拟化运行 Linux。

## 基本环境准备

以宿主机（x86-64）为 Ubuntu 操作系统环境为例，需要安装以下几款软件包（如果你顺利通过了导学阶段，这将不成问题）：

```
sudo apt update
sudo apt install opensbi qemu-system-misc u-boot-qemu
```

!!! note

    这里更推荐你从源码手动编译 QEMU，结合前面的所学实践起来。

另外我们还需要准备一个 RISC-V 版本的 Ubuntu 镜像，用于模拟 RISC-V 硬件虚拟化的使用环境。

直接在 Ubuntu 官网下载即可（请自己尝试 STFW 获取），这里推荐使用 preinstalled 版本，不需要自己手动安装操作系统。

镜像下载好以后，需要解压和扩容：

```
# 解压下载好的镜像
xz -dk ubuntu-24.04.2-preinstalled-server-riscv64.img.xz
```

```
# 镜像扩容
qemu-img resize -f raw ubuntu-24.04-preinstalled-server-riscv64.img +5G
```

使用如下命令启动 RISC-V Ubuntu 镜像：

```
qemu-system-riscv64 \
    -machine virt -nographic -m 4096 -smp 32 \
    -kernel /usr/lib/u-boot/qemu-riscv64_smode/uboot.elf \
    -device virtio-net-device,netdev=eth0 \
    -netdev user,id=eth0,hostfwd=tcp::2222-:22 \
    -device virtio-rng-pci \
    -drive file=ubuntu-25.04-preinstalled-server-riscv64.img,format=raw,if=virtio
```

下面对每个配置项进行详解：

- `-machine virt` 这里我们配置了 virt Machine 来运行客户机操作系统（virt 默认开启了对 H 扩展的支持）。

- `-nographic` 不需要图形界面，通过命令行终端打印客户机串口输出，并允许人机交互 (先按下 ctrl + a, 再按下 c 进入)。

- `-m 4096 -smp 32` 分配 4G 内存，32 个核心（u-boot 最大支持数量）。

- `-kernel /.../uboot.elf` 我们通过 U-Boot 来引导 kernel。

- `-device virtio-net-device,netdev=eth0` 高性能半虚拟化网卡（基于 VirtIO 标准），绑定到名为 eth0 的后端网络设备。

- `-netdev user,id=eth0,hostfwd=tcp::2222-:22` 使用 QEMU 内置的用户模式网络栈（无需主机网桥），并把客户机的 22 端口映射到宿主机的 2222 端口，以便通过 ssh 等远程连接工具访问虚拟机。

- `-device virtio-rng-pci` 基于 PCI 的虚拟随机数生成器（RNG），Linux 内核需加载 virtio_rng 驱动

- `-drive file=ubuntu-riscv64.img,format=raw,if=virtio` 加载磁盘镜像

!!! tip

    QEMU 启动 RISC-V 的第一阶段 boot 是 OpenSBI，在 7.0 版本以前需要通过 -bios 选项指定，后续高版本，QEMU 默认自动加载，无需手动指定。第二阶段，才是 U-Boot。

以上命令不必强制记忆，可以通过 qemu 的 help 命令查询。一般在生产环境，我们会使用脚本或者交互更友好的中间件或者上层软件来操作，比如 libvirt。

## 配置 RISC-V Ubuntu

成功启动 Ubuntu 以后，将会看到以下打印信息，我们使用默认的用户名 ubuntu 来登录，并修改初始密码 ubuntu 为你需要的密码，操作如下：

```
...
[  OK  ] Started getty@tty1.service - Getty on tty1.
[  OK  ] Reached target getty.target - Login Prompts.
Ubuntu 25.04 ubuntu ttyS0
ubuntu login: ubuntu # 输入用户名
Password: ubuntu # 输入密码，之后会提示你修改初始密码
...
Welcome to Ubuntu 25.04 (GNU/Linux 6.14.0-13-generic riscv64)
```

成功登录以后，我们简单验证一下客户机操作系统是否支持虚拟化：

```
grep isa /proc/cpuinfo 
isa             : rv64imafdch_zicbom_zicboz_ziccrse_zicntr_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zca_zcd_zba_zbb_zbc_zbs_sstc_svadu_svvptc
...
```

可以看到 32 个 CPU 的 isa 部分，都有 h 扩展的字符，这说明我们模拟的硬件是支持虚拟化扩展的。

接着配置一下环境，为了方便使用虚拟化能力来运行 Linux，我们依旧可以通过 QEMU 来实现，先安装一些必要软件包（由于开启了网络直通，可以直接安装）：

```
sudo apt update；sudo apt upgrade -y
sudo apt install qemu-system
```

我们使用 Revyos 提供的 RISC-V 虚拟化测试镜像及脚本进行验证：

```
sudo apt install -y wget # 如系统未安装 wget 工具，请先安装
wget https://mirror.iscas.ac.cn/revyos/extra/kvm_demo/rootfs_kvm_guest.img \
https://mirror.iscas.ac.cn/revyos/extra/kvm_demo/start_vm.sh \
https://mirror.iscas.ac.cn/revyos/extra/kvm_demo/Image
chmod +x start_vm.sh; ./start_vm.sh # 启动 KVM
```

成功启动以后，输出如下：

```
...
[    6.538768] Freeing unused kernel image (initmem) memory: 2200K
[    6.561827] Run /init as init process
           _  _
          | ||_|
          | | _ ____  _   _  _  _
          | || |  _ \| | | |\ \/ /
          | || | | | | |_| |/    \
          |_||_|_| |_|\____|\_/\_/

               Busybox Rootfs

Please press Enter to activate this console.
/ # uname -a
Linux (none) 6.6.30 #1 SMP Sat May 11 15:15:11 UTC 2024 riscv64 GNU/Linux
/ #
```

接下来，就可以在 QEMU 上体验 RISC-V 的硬件虚拟化扩展了。

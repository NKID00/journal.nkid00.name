+++
title = "构建最小 Linux 内核调试环境"
date = 2023-05-20T14:56:24+08:00
author = "NKID00"
tags = ["GNU/Linux", "QEMU", "Kernel", "GDB"]
hideComments = false
draft = false
+++

准备看看 Linux 内核，于是需要起一个基础的调试环境。

## 内核

先安装各种依赖（libncurses-dev 用于配置内核选项的界面，lz4 用于压缩内核，wget 用于下载源码包，bear 用于生成编译数据库）：

```sh
sudo apt install build-essential flex bison bc libelf-dev libssl-dev libncurses-dev lz4 wget bear
```

下载内核源码，这里使用最新的 6.4-rc2 版本：

```sh
wget 'https://git.kernel.org/torvalds/t/linux-6.4-rc2.tar.gz'
tar xf linux-6.4-rc2.tar.gz
cd linux-6.4-rc2
```

清理源码树，删除之前的所有配置和构建结果：

```sh
make mrproper
```

设置选项让 make 吃光多核并且在 build 目录下构建：

```sh
mkdir build
export MAKEFLAGS="-j$(nproc) O=build"
```

设置编译选项，先用默认选项加 debug：

```sh
make x86_64_defconfig
make kvm_guest.config
make x86_debug.config
```

再具体设置，会弹出一个终端图形界面和一堆选项：

```sh
make nconfig
# 或者使用真的图形界面
# make xconfig  # Qt 界面
# make gconfig  # GTK 界面
```

修改功能和驱动，因为只需要在 QEMU 里跑起来所以可以取消很多不必要的：

```
General setup  --->
    Kernel compression mode (Gzip)  --->
        <X> LZ4  # 使用 LZ4 压缩内核（夹带私货 :P）
    [ ] Initial RAM filesystem and RAM disk (initramfs/initrd) support
    [ ] Preserve cpio archive mtimes in initramfs

Processor type and features  --->
    [ ] Symmetric multi-processing support  # 虚拟机只用单核便于调试
    [ ] Enable MPS table
    [ ] Support for extended (non-PC) x86 platforms
    [ ] CPU microcode loading support
    [ ]   Randomize the address of the kernel image (KASLR)  # 取消内核地址随机化便于调试

[ ] Mitigations for speculative execution vulnerabilities  ----

Device Drivers
    < > PCCard (PCMCIA/CardBus) support  ----
    [*] Block devices  --->
        <*>   RAM block device support  # 不选这个会无法挂载 RAM 块设备而 Kernel Panic
    < > Serial ATA and Parallel ATA drivers (libata)  ----
    SCSI device support  --->
        < > SCSI device support
    [ ] Multiple devices driver support (RAID and LVM)  ----
    [ ] Macintosh device drivers  ----
    [*] Network device support  --->
        < >     Network console logging support
        [ ]   Ethernet driver support  ----
        < >   PHY Device support and infrastructure  ----
        < >   MDIO bus device drivers  ----
        < >   USB Network Adapters  ----
        [ ]   Wireless LAN  ----
    [ ] Watchdog Timer Support  ----
    Graphics support  ---
        < > /dev/agpgart (AGP Support)  ----
        < > Direct Rendering Manager (XFree86 4.1.0 and higher DRI support)  ----
    < > Sound card support  ----
    [ ] HID bus support  ----
    [ ] USB support  ----
    [*] Virtio drivers  --->
        [ ]     Support for legacy virtio draft 0.9.X and older devices
    [ ] Microsoft Surface Platform-Specific Device Drivers  ----

[*] Networking support  --->
    [ ]   Wireless  ----

Security options  --->
    [ ] Socket and Networking Security Hooks
    [ ] NSA SELinux Support
    [ ] Integrity subsystem

# 取消 ext4 支持，启用 Btrfs 支持（继续夹带私货 :P）
File systems  --->
    < > The Extended 4 (ext4) filesystem
    <*> Btrfs filesystem support
    [*]   Btrfs POSIX Access Control Lists
    CD-ROM/DVD Filesystems  --->
        < > ISO 9660 CDROM file system support
    DOS/FAT/EXFAT/NT Filesystems  --->
        < > MSDOS fs support
        < > VFAT (Windows-95) fs support
    [ ] Miscellaneous filesystems  ----
    [ ] Network File Systems  ----
```

编译，顺便生成 clangd 需要的编译数据库 `compile_commands.json`（因为前面删了很多选项所以挺快的，R7 4800U -j16 花了大概 5 分钟）：

```sh
bear -- make all
```

生成 GDB 辅助脚本：

```sh
make scripts_gdb
```

## Clangd

内核会用到大量 GNU C 语言扩展和一些 clangd 不支持的编译选项，所以需要在源码树根目录下加一个 `.clangd` 文件填入以下内容消除报错：

```yaml
CompileFlags:
  Add:
  - -Wno-gnu
  Remove: 
  - -mpreferred-stack-boundary=3
  - -mindirect-branch=thunk-extern
  - -mindirect-branch-register
  - -mindirect-branch-cs-prefix
  - -mfunction-return=thunk-extern
  - -fno-allow-store-data-races
  - -fconserve-stack
  - -mrecord-mcount
  - -ftrivial-auto-var-init=zero
```

然后就可以愉快地用 clangd 看代码了，好耶！

## 根目录

懒得折腾直接用 Alpine Linux 提供的 rootfs，这里使用最新的 3.18.0 版本：

```sh
cd ..
wget 'https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/x86_64/alpine-minirootfs-3.18.0-x86_64.tar.gz'
wget 'https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/x86_64/alpine-minirootfs-3.18.0-x86_64.tar.gz.asc'
wget 'https://alpinelinux.org/keys/ncopa.asc'  # 数字签名对应的公钥
gpg --import ncopa.asc
gpg --verify alpine-minirootfs-3.18.0-x86_64.tar.gz.asc alpine-minirootfs-3.18.0-x86_64.tar.gz
```

确认出现下面的信息表明签名正确（这里的 warning 是预期的）：

```
gpg: Signature made Wed 10 May 2023 03:51:41 AM CST
gpg:                using RSA key 0482D84022F52DF1C4E7CD43293ACD0907D9495A
gpg: Good signature from "Natanael Copa <ncopa@alpinelinux.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 0482 D840 22F5 2DF1 C4E7  CD43 293A CD09 07D9 495A
```

创建 rootfs 镜像，大小可以随意更改，这里使用前面编译进内核的 Btrfs：

```sh
fallocate -l 1GiB rootfs.img
mkfs.btrfs rootfs.img
```

挂载镜像：

```sh
sudo mkdir -p /mnt/rootfs
sudo mount rootfs.img /mnt/rootfs
```

然后把 Alpine Linux 的 rootfs 解压进去：

```sh
sudo tar xf alpine-minirootfs-3.18.0-x86_64.tar.gz -C /mnt/rootfs
```

chroot 进入镜像进行配置：

```sh
sudo chroot /mnt/rootfs /bin/sh -l
```

配置网络：

```sh
echo 'alpine' >/etc/hostname  # 任意 hostname
echo 'nameserver 223.5.5.5' >/etc/resolv.conf  # 任意 DNS 服务器
(cat <<EOF
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF
) >/etc/network/interfaces
```

安装包：

```sh
apk update
apk upgrade
apk add openrc util-linux
```

添加服务，启用串口终端和自动登录，自动挂载 sysfs：

```sh
rc-update add networking
rc-update add hostname
sed -i -r -e 's/#ttyS0.*/ttyS0::respawn:\/sbin\/agetty --autologin root --noclear ttyS0 115200 vt100/' /etc/inittab
echo 'sysfs /sys sysfs rw,nosuid,nodev,noexec,relatime 0 0' >>/etc/fstab
```

退出 chroot，卸载镜像（非常重要！文件系统同时被两个系统挂载会出事）：

```sh
^D
sudo umount /mnt/rootfs
```

## QEMU

apt 嗯装：

```sh
sudo apt install qemu-system
```

启动：

```sh
sudo qemu-system-x86_64 \
  -enable-kvm -cpu host -m 1G -nographic  `# 启用 KVM，单核，1G 内存，无图形界面` \
  -kernel linux-6.4-rc2/build/arch/x86_64/boot/bzImage \
  -append "root=/dev/vda rw console=ttyS0 tsc=reliable"  `# 内核参数：指定根目录，串口终端，强制设定时钟源` \
  -drive file=rootfs.img,format=raw,if=virtio  `# 根目录` \
  -nic user,model=virtio  `# 网卡` \
  -s  `# 在端口 1234 上启用 GDB 远程调试`

# 或者不使用 kvm
sudo qemu-system-x86_64 \
  -cpu max -m 1G -nographic  `# 单核，1G 内存，无图形界面` \
  -kernel linux-6.4-rc2/build/arch/x86_64/boot/bzImage \
  -append "root=/dev/vda rw console=ttyS0 tsc=reliable"  `# 内核参数：指定根目录，串口终端，强制设定时钟源` \
  -drive file=rootfs.img,format=raw,if=virtio  `# 根目录` \
  -nic user,model=virtio  `# 网卡` \
  -s  `# 在端口 1234 上启用 GDB 远程调试`
```

令人安心的 Alpine Linux shell 很快会弹出来。

## GDB

```sh
sudo apt install gdb
```

允许自动加载辅助脚本：

```sh
echo "add-auto-load-safe-path $(realpath linux-6.4-rc2/build/vmlinux-gdb.py)" >> ~/.gdbinit
```

启动 GDB 加载内核调试符号并附加远程调试：

```sh
gdb -ex 'target remote :1234' linux-6.4-rc2/build/vmlinux
```

然后就可以开始调试了捏。

坑：如果启动虚拟机时启用了 KVM，调试时可能会来中断，但不启用 KVM 则很慢。

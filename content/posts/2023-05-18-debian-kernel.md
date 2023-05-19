+++
title = "在 Debian 上编译最新的 Linux 内核"
date = 2023-05-18T15:49:11+08:00
author = "NKID00"
tags = ["GNU/Linux", "Debian", "Kernel"]
hideComments = false
draft = false
+++

最近搞到了一台没有运行服务可以随意折腾的 4C8G 测试机，安装了 Debian sid 之后没活整了，于是打算更新下内核。编译内核大约需要 10G 硬盘空间。

## 准备

安装各种依赖（注意这里编译的是云原生内核，普通的内核需要的依赖更多，wget 只是为了下载内核源码包）：

```sh
apt install build-essential flex bison bc debhelper rsync libelf-dev libssl-dev pahole lz4 \
            wget
```

下载最新的内核源码，解压并校验（这里用的是 6.3.3 版本的内核）：

```sh
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.3.3.tar.xz
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.3.3.tar.sign
wget https://git.kernel.org/pub/scm/docs/kernel/pgpkeys.git/plain/keys/38DBBDC86092693E.asc  # 数字签名对应的公钥
gpg --import 38DBBDC86092693E.asc
unxz linux-6.3.3.tar.xz
gpg --verify linux-6.3.3.tar.sign linux-6.3.3.tar
```

确认出现下面的信息表明签名正确（这里的 warning 是预期的）：

```
gpg: Signature made Wed 17 May 2023 08:03:56 PM CST
gpg:                using RSA key 647F28654894E3BD457199BE38DBBDC86092693E
gpg: Good signature from "Greg Kroah-Hartman <gregkh@linuxfoundation.org>" [unknown]
gpg:                 aka "Greg Kroah-Hartman (Linux kernel stable release signing key) <greg@kroah.com>" [unknown]
gpg:                 aka "Greg Kroah-Hartman <gregkh@kernel.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 647F 2865 4894 E3BD 4571  99BE 38DB BDC8 6092 693E
```

然后解包，需要确保当前目录可写，最终的 deb 包会被生成到源码树的上一级目录而非源码树所在目录：

```sh
tar xf linux-6.3.3.tar
cd linux-6.3.3
```

## 构建

设置选项让 make 吃光多核：

```sh
export MAKEFLAGS="-j$(nproc)"
```

清理源码树，删除之前的所有配置和构建结果：

```sh
make mrproper
```

复制当前系统使用的内核选项作为模板：

```sh
cp /boot/config-6.1.0-9-cloud-amd64 .config  # 这里的 6.1.0-9-cloud-amd64 是当前的内核版本
```

根据需求更新内核选项（懒得折腾就直接用默认值 :P ）：

```sh
make oldconfig  # 手动指定每个新出现的选项
make olddefconfig  # 对新出现的选项直接应用默认值
```

激动人心的编译环节终于到了（这一步会非常慢，在测试机上大概花了一个小时）：

```sh
make bindeb-pkg
```

构建结束后在上一级目录中会生成这些 deb 包和其他一些信息文件：

```
linux-headers-6.3.3_6.3.3-6_amd64.deb
linux-image-6.3.3-dbg_6.3.3-6_amd64.deb
linux-image-6.3.3_6.3.3-6_amd64.deb
linux-libc-dev_6.3.3-6_amd64.deb
```

## 安装

```sh
cd ..
dpkg -i linux-image-6.3.3_6.3.3-6_amd64.deb linux-libc-dev_6.3.3-6_amd64.deb
```

重启。完成！

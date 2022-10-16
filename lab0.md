# Exercise 0 实验环境搭建
[TOC]

所有实验均基于Linux操作系统进行开发，可以使用云服务器或虚拟机完成实验。

## Part 1 Rust 开发环境配置

### 1.1 普通安装Rust

首先安装 Rust 版本管理器 rustup 和 Rust 包管理器 cargo，这里我们用官方的安装脚本来安装：

```
curl https://sh.rustup.rs -sSf | sh
```

如果通过官方的脚本下载失败了，可以在浏览器的地址栏中输入 [https://sh.rustup.rs](https://sh.rustup.rs/) 来下载脚本，在本地运行即可。

### 1.2 换源安装Rust

如果官方的脚本在运行时出现了网络速度较慢的问题。可以通过修改 rustup 的镜像地址（修改为中国科学技术大学的镜像服务器）来加速。从上之下依次执行下列命令：

1.回到根目录

```bash
cd ~ 
```

2.编辑.bashrc 修改环境变量

```bash
vim .bashrc
```

```bash
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
```

或者使用tuna源来加速 [参见 rustup 帮助](https://mirrors.tuna.tsinghua.edu.cn/help/rustup/)：

```bash
export RUSTUP_DIST_SERVER=https://mirrors.tuna.edu.cn/rustup
export RUSTUP_UPDATE_ROOT=https://mirrors.tuna.edu.cn/rustup/rustup
```

或者也可以通过在运行前设置命令行中的科学上网代理来实现：

```bash
# e.g. Shadowsocks 代理，请根据自身配置灵活调整下面的链接
export https_proxy=http://127.0.0.1:1080
export http_proxy=http://127.0.0.1:1080
export ftp_proxy=http://127.0.0.1:1080
```

3.通过如下命令安装rust。

```bash
apt install curl vim gcc
curl https://sh.rustup.rs -sSf | sh
```

4.操作系统实验依赖于nightly版本的rust，因此需要安装nightly版本，并将rust默认设置为nightly。
执行如下命令：

```bash
rustup install nightly
rustup default nightly
```

接下来，我们可以确认一下我们正确安装了 Rust 工具链：

```bash
rustc --version
```

可以看到当前安装的工具链的版本。

```
rustc 1.66.0-nightly (c0983a9aa 2022-10-12)
```

### 1.3 修改Cargo的源

我们打开（如果没有就新建） `~/.cargo/config` 文件，并把内容修改为：

```
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"
```

同样，也可以使用tuna源 [参见 crates.io 帮助](https://mirrors.tuna.tsinghua.edu.cn/help/crates.io-index.git/)：

```
[source.crates-io]
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```

接下来安装一些Rust相关的软件包

```bash
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils --vers =0.3.3
rustup component add llvm-tools-preview
rustup component add rust-src
```

## Part 2 QEMU 模拟器安装

[QEMU](https://zh.m.wikipedia.org/zh-hans/QEMU),简单而言是一个硬件虚拟化的仿真程序，它本质上是一个托管的虚拟机，通过动态二进制转换，模拟CPU，并且提供一组设备模型，使它能够运行多种未修改的客户机操作系统。

执行如下命令安装基本的软件包：

```bash
apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev git tmux python3 ninja-build wget
```

然后通过源码安装qemu 5.2。

第一步下载:

```bash
wget https://download.qemu.org/qemu-5.2.0.tar.xz
```

第二步解包:

```
tar xvJf qemu-5.2.0.tar.xz
```

第三步安装：

```bash
cd qemu-5.2.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user
make -j$(nproc) install
```

安装完成后可以通过如下命令验证qemu是否安装成功。

```bash
qemu-system-riscv64 --version
qemu-riscv64 --version
```

```
QEMU emulator version 5.2.0
Copyright (c) 2003-2020 Fabrice Bellard and the QEMU Project developers
root@iZuf6275hb357m7muv14wsZ:~# qemu-riscv64 --version
qemu-riscv64 version 5.2.0
Copyright (c) 2003-2020 Fabrice Bellard and the QEMU Project developers
```


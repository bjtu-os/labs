# Lab 0 实验环境搭建

所有实验均基于Linux操作系统进行开发，可以使用云服务器或本地虚拟机完成实验。

## C 开发环境配置

在实验或练习过程中，也会涉及部分基于C语言的开发，可以安装基本的本机开发环境和交叉开发环境。下面是以Ubuntu 20.04为例，需要安装的C 开发环境涉及的软件：

```shell
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

## Rust 开发环境配置

首先安装 Rust 版本管理器 rustup 和 Rust 包管理器 cargo，这里我们用官方的安装脚本来安装：

```shell
curl https://sh.rustup.rs -sSf | sh
```

如果通过官方的脚本下载失败了，可以在浏览器的地址栏中输入 [https://sh.rustup.rs](https://sh.rustup.rs/) 来下载脚本，在本地运行即可。

如果官方的脚本在运行时出现了网络速度较慢的问题，可选地可以通过修改 rustup 的镜像地址（修改为清华大学的镜像服务器）来加速 [参见 rustup 帮助](https://mirrors.tuna.tsinghua.edu.cn/help/rustup/)：

```shell
export RUSTUP_DIST_SERVER=https://mirrors.tuna.edu.cn/rustup
export RUSTUP_UPDATE_ROOT=https://mirrors.tuna.edu.cn/rustup/rustup
curl https://sh.rustup.rs -sSf | sh
```

安装完成后，我们可以重新打开一个终端来让之前设置的环境变量生效。我们也可以手动将环境变量设置应用到当前终端，只需要输入以下命令：

```shell
source $HOME/.cargo/env
```

接下来，我们可以确认一下我们正确安装了 Rust 工具链：

```shell
rustc --version
```

可以看到当前安装的工具链的版本。

```shell
rustc 1.62.0-nightly (1f7fb6413 2022-04-10)
```

可通过如下命令安装 rustc 的 nightly 版本，并把该版本设置为 rustc 的缺省版本。

```shell
rustup install nightly
rustup default nightly
```

我们最好把软件包管理器 cargo 所用的软件包镜像地址 crates.io 也换成清华大学的镜像服务器来加速三方库的下载。我们打开（如果没有就新建） `~/.cargo/config` 文件，并把内容修改为 [参见 crates.io 帮助](https://mirrors.tuna.tsinghua.edu.cn/help/crates.io-index.git/)：

```toml
[source.crates-io]
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```

接下来安装一些Rust相关的软件包

```shell
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils --vers =0.3.3
rustup component add llvm-tools-preview
rustup component add rust-src
```

至于 Rust 开发环境，推荐 JetBrains Clion + Rust插件 或者 Visual Studio Code 搭配 rust-analyzer 和 RISC-V Support 插件。

> - JetBrains Clion是付费商业软件，但对于学生和教师，只要在 JetBrains 网站注册账号，可以享受一定期限（半年左右）的免费使用的福利。
> - Visual Studio Code 是开源软件，不用付费就可使用。
> - 当然，采用 VIM，Emacs 等传统的编辑器也是没有问题的。

## QEMU 模拟器安装

[QEMU](https://zh.m.wikipedia.org/zh-hans/QEMU)，简单而言是一个硬件虚拟化的仿真程序，它本质上是一个托管的虚拟机，通过动态二进制转换，模拟CPU，并且提供一组设备模型，使它能够运行多种未修改的客户机操作系统。

执行如下命令安装基本的软件包：

```bash
apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev git tmux python3 ninja-build wget
```

然后通过源码安装qemu。

第一步，下载:

```bash
wget https://download.qemu.org/qemu-7.0.0.tar.xz
```

第二步，解压：

```shell
tar xvJf qemu-7.0.0.tar.xz
```

第三步，编译安装并配置 RISC-V 支持：

```bash
cd qemu-7.0.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user  # 如果要支持图形界面，可添加 " --enable-sdl" 参数
make -j$(nproc)
```

之后我们可以在同目录下 `sudo make install` 将 QEMU 安装到 `/usr/local/bin` 目录下，但这样经常会引起冲突。个人来说更习惯的做法是，编辑 `~/.bashrc` 文件（如果使用的是默认的 `bash` 终端），在文件的末尾加入几行：

```shell
# 请注意，qemu-7.0.0 的父目录可以随着你的实际安装位置灵活调整
export PATH=$PATH:/path/to/qemu-7.0.0/build
```

随后即可在当前终端 `source ~/.bashrc` 更新系统路径，或者直接重启一个新的终端。

此时我们可以确认 QEMU 的版本：

```shell
qemu-system-riscv64 --version
qemu-riscv64 --version
```


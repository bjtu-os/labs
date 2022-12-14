# Lab 0 实验环境搭建

**注意**： 本教程是基于 **openEuler 22.03** 操作系统来进行实验的。如果使用 **Ubuntu** 操作系统的话，注意包管理器需要使用`apt`，安装相应包的指令也会发生变化。但是后续的实验步骤和 **openEuler** 并无太大的区别。

## 下载 openEuler 操作系统镜像

**openEuler 22.03 LTS** 操作系统镜像的官网地址：https://repo.openeuler.org/openEuler-22.03-LTS/ISO/

- x86 CPU版本地址：https://repo.openeuler.org/openEuler-22.03-LTS/ISO/x86_64/openEuler-22.03-LTS-x86_64-dvd.iso
- ARM CPU版本地址：https://repo.openeuler.org/openEuler-22.03-LTS/ISO/aarch64/openEuler-22.03-LTS-aarch64-dvd.iso

## 安装镜像

首先需要在你的电脑上安装虚拟机，如果已经安装过了就可以跳过此步。

- Windows: 推荐使用 VMware Workstation
- macOS: 更推荐使用 Parallels Desktop，VMware Fusion 也可以

**注意**：VirtualBox 虚拟机会在后续实验操作中出现问题，不建议使用。

在安装 openEuler 操作系统镜像的过程中，会显示一个图形用户界面来让你进行基础信息的配置，在配置用户信息时，要选择 **拥有管理员权限**，如果不选也可以，但后续调试会比较麻烦。

## 基础软件环境配置

在实验过程中，会用到一些基本的软件包：

```shell
$ sudo dnf groupinstall "Development Tools"
$ sudo dnf install autoconf automake gcc gcc-c++ kernel-devel curl libmpc-devel \
mpfr-devel gmp-devel glib2 glib2-devel make cmake gawk bison flex texinfo gperf libtool \
patchutils bc python3 ninja-build wget xz curl gcc vim
```

## Rust 开发环境配置

经过多次测试，如果使用国内镜像安装的话会出现各种各样的问题，所以我们用官方的安装脚本来安装 Rust 版本管理器 `rustup` 和 Rust 包管理器 `cargo`，如果速度慢就开 VPN：

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

安装完成后，我们手动将环境变量设置应用到当前终端，只需要输入以下命令：

```shell
source "$HOME/.cargo/env"
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

我们最好把软件包管理器 `cargo` 所用的软件包镜像地址 crates.io 也换成清华大学的镜像服务器来加速三方库的下载。 [参见 crates.io 帮助](https://mirrors.tuna.tsinghua.edu.cn/help/crates.io-index.git/)：

首先使用该命令编辑 `~/.cargo/config` 文件：

```shell
vim ~/.cargo/config
```

按 `i` 键进入编辑模式，复制粘贴以下内容：

```toml
[source.crates-io]
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```

按 `esc` 键退出编辑模式，再按 `:wq` 加 **回车** 就可以保存并退出文件了，如果有同学之前没接触过 vim 编辑器，建议系统性学习一下。

接下来安装一些 Rust 相关的软件包：

```shell
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils --vers =0.3.3
rustup component add llvm-tools-preview
rustup component add rust-src
```

至于 Rust 开发环境，推荐 JetBrains CLion + Rust 插件 或者 Visual Studio Code 搭配 rust-analyzer 和 RISC-V Support 插件。

> - JetBrains CLion 是付费商业软件，但对于学生和教师，只要在 JetBrains 官网申请 Educational License，可以享受一定期限（一年左右）的免费使用的福利。
> - Visual Studio Code 是开源软件，不用付费就可使用。
> - 当然，采用 VIM，Emacs 等传统的编辑器也是没有问题的。

## QEMU 模拟器安装

[QEMU](https://en.wikipedia.org/wiki/QEMU) 是一个托管的虚拟机，它通过动态的二进制转换，模拟 CPU，并且提供一组设备模型，使它能够运行多种未修改的客户机 OS，可以通过与 [KVM](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine) 一起使用进而接近本地速度运行虚拟机（接近真实电脑的速度）。我们现在通过源码安装 QEMU：

```shell
# 下载源码压缩包
wget https://download.qemu.org/qemu-7.0.0.tar.xz

# 解压
tar xvJf qemu-7.0.0.tar.xz

# 编译安装并配置 RISC-V 支持：
cd qemu-7.0.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user  # 如果要支持图形界面，可添加 " --enable-sdl" 参数
make -j $(nproc)
```

之后我们可以在同目录下 `sudo make install` 将 QEMU 安装到 `/usr/local/bin` 目录下，但这样经常会引起冲突。个人来说更习惯的做法是，编辑 `~/.bashrc` 文件（如果使用的是默认的 `bash` 终端），在文件的末尾加入几行：

```shell
# 请注意，qemu-7.0.0 的父目录需要随着你的实际安装位置灵活调整
export PATH=$PATH:/path/to/qemu-7.0.0/build
```

随后即可在当前终端 `source ~/.bashrc` 更新系统路径，或者直接重启一个新的终端。

此时我们可以确认 QEMU 的版本：

```shell
qemu-system-riscv64 --version
qemu-riscv64 --version
```

至此，Lab 0 就已完成了。

## Appendix

### 使用 VS Code Remote SSH 连接

openEuler 默认关闭了 `AllowTcpForwarding` 选项，如果需要使用 VS Code Remote SSH 的话，需要先在 openEuler 系统上的 `/etc/ssh/sshd_config` 最底下找到 `AllowTcpForwarding no` 这一行删掉并保存，然后重启 `sshd` 服务：

```shell
sudo service ssh restart
```

### 关于如何在 Linux 环境下复制粘贴

首先在虚拟机下使用该命令查看本虚拟机的ip地址：

```shell
$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:1c:42:b1:0c:c2 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.6/24 brd 10.211.55.255 scope global dynamic noprefixroute enp0s5
       valid_lft 1002sec preferred_lft 1002sec
    inet6 fdb2:2c26:f4e4:0:2fc4:c989:498e:5519/64 scope global dynamic noprefixroute
       valid_lft 2591648sec preferred_lft 604448sec
    inet6 fe80::3a0b:9181:b18:2072/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

注意这里能看出来本地虚拟机的ip地址是 `10.211.55.6`，然后打开自己电脑的终端界面，在**保持虚拟机运行的状态下**，输入该命令登录虚拟机：

```shell
ssh root@10.211.55.6
```

然后在自己的终端界面就可以自由使用复制粘贴功能了。

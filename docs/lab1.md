# Lab 1 独立可执行程序

## 本节导读

本节我们从一个最简单的 Rust 应用程序入手，不仅仅满足于停留在它的表面，而是深入地挖掘它下面的多层执行环境为它的开发和运行提供了怎样的方便。特别的，操作系统也是一层执行环境，因此它需要为上层的应用程序提供服务。我们会接触到计算机科学中最核心的思想——抽象，同时以现实中不同需求层级的应用为例分析如何进行合理的抽象。最后，我们还会介绍软硬件平台（包括 RISC-V 架构）的一些基础知识。

## 执行应用程序

我们先在Linux上开发并运行一个简单的 “Hello, world” 应用程序，看看一个简单应用程序从开发到执行的全过程。作为一切的开始，让我们使用 Cargo 工具来创建一个 Rust 项目。它看上去没有任何特别之处：

```bash
$ cargo new os --bin
```

我们加上了 `--bin` 选项来告诉 Cargo 我们创建一个可执行程序项目而不是函数库项目。此时，项目的文件结构如下：

```bash
$ tree os
os
├── Cargo.toml
└── src
    └── main.rs

1 directory, 2 files
```

其中 `Cargo.toml` 中保存着项目的配置，包括作者的信息、联系方式以及库依赖等等。显而易见源代码保存在 `src` 目录下，目前为止只有 `main.rs` 一个文件，让我们看一下里面的内容：

```rust
fn main() {
    println!("Hello, world!");
}
```

进入 os 项目根目录下，利用 Cargo 工具即可一条命令实现构建并运行项目：

```bash
$ cargo run
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
    Finished dev [unoptimized + debuginfo] target(s) in 1.15s
     Running `target/debug/os`
Hello, world!
```

如我们预想的一样，我们在屏幕上看到了一行 `Hello, world!` 。但是，需要注意到**我们所享受到的编程和执行程序的方便性并不是理所当然的，背后有着从硬件到软件的多种机制的支持。**简而言之，我们使用了操作系统和Rust语言本身提供的许多便捷接口。我们在打印 `Hello, world!` 时使用的 `println!` 宏正是由 Rust 标准库 std 和 GNU Libc 库等提供的。

从内核/操作系统的角度看来，它上的上层软件的一切操作都属于用户态，而操作系统自身属于内核态。无论用户态应用如何编写，是手写汇编代码，还是基于某种高级编程语言调用其标准库或三方库，某些功能总要直接或间接的通过内核/操作系统提供的 **系统调用** (System Call) 来实现。因此系统调用充当了用户和内核之间的边界。内核作为用户态的执行环境，它不仅要提供系统调用接口，还需要对用户态应用的执行进行监控和管理。

从硬件的角度来看，它上面的一切都属于软件。硬件可以分为三种： 处理器 (Processor，也称CPU)，内存 (Memory) 还有 I/O 设备。其中处理器无疑是其中最复杂，同时也最关键的一个。它与软件约定一套 **指令集体系结构** (ISA, Instruction Set Architecture)，使得软件可以通过 ISA 中提供的机器指令来访问各种硬件资源。软件当然也需要知道处理器会如何执行这些指令：最简单的话就是一条一条执行位于内存中的指令。当然，实际的情况远比这个要复杂得多，为了适应现代应用程序的场景，处理器还需要提供很多额外的机制（如特权级、页表、TLB、异常/中断响应等），而不仅仅是让数据在 CPU 寄存器、内存和 I/O 设备三者之间流动。

> 计算机科学中遇到的所有问题都可通过增加一层抽象来解决。
>
> All problems in computer science can be solved by another level of indirection。
>
> – 计算机科学家 David Wheeler

## 平台与目标三元组

编译器在编译、链接得到可执行文件时需要知道，程序要在哪个 **平台** (Platform) 上运行， **目标三元组** (Target Triplet) 描述了目标平台的 CPU 指令集、操作系统类型和标准运行时库。

我们研究一下现在 `Hello, world!` 程序的目标三元组是什么：

```shell
$ rustc --version --verbose
   rustc 1.61.0-nightly (68369a041 2022-02-22)
   binary: rustc
   commit-hash: 68369a041cea809a87e5bd80701da90e0e0a4799
   commit-date: 2022-02-22
   host: x86_64-unknown-linux-gnu
   release: 1.61.0-nightly
   LLVM version: 14.0.0
```

其中 host 一项表明默认目标平台是 `x86_64-unknown-linux-gnu`， CPU 架构是 x86_64，CPU 厂商是 unknown，操作系统是 linux，运行时库是 gnu libc。

接下来，我们希望把 `Hello, world!` 移植到 RICV 目标平台 `riscv64gc-unknown-none-elf` 上运行。

> `riscv64gc-unknown-none-elf` 的 CPU 架构是 riscv64gc，厂商是 unknown，操作系统是 none， elf 表示没有标准的运行时库。没有任何系统调用的封装支持，但可以生成 ELF 格式的执行程序。 我们不选择有 linux-gnu 支持的 `riscv64gc-unknown-linux-gnu`，是因为我们的目标是开发操作系统内核，而非在 linux 系统上运行的应用程序。

## 修改目标平台

将程序的目标平台换成 `riscv64gc-unknown-none-elf`，试试看会发生什么：

```shell
$ cargo run --target riscv64gc-unknown-none-elf
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
error[E0463]: can't find crate for `std`
  |
  = note: the `riscv64gc-unknown-none-elf` target may not be installed
```

报错的原因是目标平台上确实没有 Rust 标准库 std，也不存在任何受 OS 支持的系统调用。 这样的平台被我们称为 **裸机平台** (bare-metal)。

幸运的是，除了 std 之外，Rust 还有一个不需要任何操作系统支持的核心库 core， 它包含了 Rust 语言相当一部分核心机制，可以满足本门课程的需求。 有很多第三方库也不依赖标准库 std，而仅仅依赖核心库 core。

为了以裸机平台为目标编译程序，我们要将对标准库 std 的引用换成核心库 core。

## 移除标准库依赖

由于后续实验需要 `rustc` 编译器缺省生成RISC-V 64的目标代码，所以我们首先要给 `rustc` 添加一个target : `riscv64gc-unknown-none-elf` 。这可通过如下命令来完成：

```
$ rustup target add riscv64gc-unknown-none-elf
```

然后在 `os` 目录下新建 `.cargo` 目录，并在这个目录下创建 `config` 文件，输入如下内容：

```
# os/.cargo/config
[build]
target = "riscv64gc-unknown-none-elf"
```

这将使 cargo 工具在 os 目录下默认会使用 riscv64gc-unknown-none-elf 作为目标平台。 这种编译器运行的平台（x86_64）与可执行文件运行的目标平台不同的情况，称为 **交叉编译** (Cross Compile)。

操作系统是很底层的应用软件，它不应该依赖Rust的标准库，因为标准库本身需要操作系统本身支持。

## 移除println!()宏

`println!` 宏所在的 Rust 标准库 std 需要通过系统调用获得操作系统的服务，而如果要构建运行在裸机上的操作系统，就不能再依赖标准库了。所以我们第一步要尝试移除 `println!` 宏及其所在的标准库。

我们在 `main.rs` 的开头加上一行 `#![no_std]` 来告诉 Rust 编译器不使用 Rust 标准库 std ，转而使用核心库 core（core库不需要操作系统的支持）。**同时记得注释掉之前使用的println!()宏**，重新编译执行，你会得到一个报错的程序，出错信息大概长这样：

```
`#[panic_handler]` function required, but not found
```

在使用 Rust 编写应用程序的时候，我们常常在遇到了一些无法恢复的致命错误（panic），导致程序无法继续向下运行。这时手动或自动调用 `panic!` 宏来打印出错的位置，让我们能够意识到它的存在，并进行一些后续处理。

Rust编译器在编译程序时，从安全性考虑，需要有 `panic!` 宏的具体实现。

在标准库 std 中提供了关于 `panic!` 宏的具体实现，其大致功能是打印出错位置和原因并杀死当前应用。

但是由于我们人工删掉了标准库依赖，导致这个宏失效了，因此我们需要手动实现一下这个宏。

> `#[panic_handler]` 是一种编译指导属性，用于标记核心库core中的 `panic!` 宏要对接的函数（该函数实现对致命错误的具体处理）。该编译指导属性所标记的函数需要具有 `fn(&PanicInfo) -> !` 函数签名，函数可通过 `PanicInfo` 数据结构获取致命错误的相关信息。这样Rust编译器就可以把核心库core中的 `panic!` 宏与 `#[panic_handler]` 指向的panic函数实现合并在一起，使得no_std程序具有类似std库的应对致命错误的功能。

我们创建一个新的子模块 `lang_items.rs` 实现panic函数，并通过 `#[panic_handler]` 属性通知编译器用panic函数来对接 `panic!` 宏。为了将该子模块添加到项目中，我们还需要在 `main.rs` 的 `#![no_std]` 的下方加上 `mod lang_items;` ，相关知识可参考 [Rust 模块编程](#Rust-模块编程) ：

```rust
// os/src/lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

在把 `panic_handler` 配置在单独的文件 `os/src/lang_items.rs` 后，需要在os/src/main.rs文件中添加以下内容才能正常编译整个软件：

```rust
// os/src/main.rs
#![no_std]
mod lang_items;
// ... other code
```

这次出错的原因是：

```shell
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
error: requires `start` lang_item
```

## 移除main函数

编译器提醒我们缺少一个名为 `start` 的语义项。

我们回忆一下，Rust语言标准库和三方库作为应用程序的执行环境需要负责在执行应用程序之前进行一些初始化工作，然后才跳转到应用程序的入口点（也就是跳转到我们编写的 `main` 函数）开始执行。

事实上 `start` 语义项代表了标准库 std 在**执行应用程序之前**需要进行的一些**初始化工作**。由于我们禁用了标准库，编译器也就找不到这项功能的实现了。

最简单的解决方案就是压根不让编译器使用这项功能。我们在 `main.rs` 的开头加入设置 `#![no_main]` 告诉编译器我们没有一般意义上的 `main` 函数，并将原来的 `main` 函数删除，压根不让编译器使用这项功能。在失去了 `main` 函数的情况下，编译器也就不需要完成所谓的初始化工作了。

至此，我们成功移除了标准库的依赖，并完成了构建裸机平台上的操作系统的第一步工作–通过编译器检查并生成执行码。

```shell
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s
```

目前的主要代码包括 `main.rs` 和 `lang_items.rs` ，大致内容如下：

```rust
// os/src/main.rs
#![no_main]
#![no_std]
mod lang_items;
// ... other code


// os/src/lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

## 分析被移除标准库的程序

对于上面这个被移除标准库的应用程序，通过了编译器的检查和编译，形成了二进制代码。但这个二进制代码是怎样的，它能否被正常执行呢？为了分析这些程序，首先需要安装 cargo-binutils 工具集：

```shell
$ cargo install cargo-binutils
$ rustup component add llvm-tools-preview
```

这样我们可以通过各种工具来分析目前的程序：

```shell
# 文件格式
$ file target/riscv64gc-unknown-none-elf/debug/os
target/riscv64gc-unknown-none-elf/debug/os: ELF 64-bit LSB executable, UCB RISC-V, ......

# 文件头信息
$ rust-readobj -h target/riscv64gc-unknown-none-elf/debug/os
   File: target/riscv64gc-unknown-none-elf/debug/os
   Format: elf64-littleriscv
   Arch: riscv64
   AddressSize: 64bit
   ......
   Type: Executable (0x2)
   Machine: EM_RISCV (0xF3)
   Version: 1
   Entry: 0x0
   ......
   }

# 反汇编导出汇编程序
$ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
   target/riscv64gc-unknown-none-elf/debug/os:       file format elf64-littleriscv
```

通过 `file` 工具对二进制程序 `os` 的分析可以看到它好像是一个合法的 RISC-V 64 可执行程序，但通过 `rust-readobj` 工具进一步分析，发现它的入口地址 Entry 是 `0` ，从 C/C++ 等语言中得来的经验告诉我们， `0` 一般表示 NULL 或空指针，因此等于 `0` 的入口地址看上去无法对应到任何指令。再通过 `rust-objdump` 工具把它反汇编，可以看到没有生成汇编代码。所以，我们可以断定，这个二进制程序虽然合法，但它是一个空程序。产生该现象的原因是：目前我们的程序（参考上面的源代码）没有进行任何有意义的工作，由于我们移除了 `main` 函数并将项目设置为 `#![no_main]` ，它甚至没有一个传统意义上的入口点（即程序首条被执行的指令所在的位置），因此 Rust 编译器会生成一个空程序。

## 总结

本小节我们固然脱离了标准库，通过了编译器的检验，但也是伤筋动骨，将原有的很多功能弱化甚至直接删除，看起来距离在 RV64GC 平台上打印 `Hello world!` 相去甚远了（我们甚至连 `println!` 和 `main` 函数都删除了）。不要着急，从下一节开始，我们将着手实现本节移除的、由用户态执行环境提供的功能。

## Appendix

### 参考资料

1. Rust 语言圣经 (格式化输出): https://course.rs/basic/formatted-output.html
2. 关于 Rust 宏的介绍: https://kaisery.github.io/trpl-zh-cn/ch19-06-macros.html
3. `file` 命令: https://www.runoob.com/linux/linux-comm-file.html
4. ELF文件格式: https://xinqiu.gitbooks.io/linux-inside-zh/content/Theory/linux-theory-2.html

### Rust 模块编程

将一个软件工程项目划分为多个子模块分别进行实现是一种被广泛应用的编程技巧，它有助于促进复用代码，并显著提升代码的可读性和可维护性。因此，众多编程语言均对模块化编程提供了支持，Rust 语言也不例外。

每个通过 Cargo 工具创建的 Rust 项目均是一个模块，取决于 Rust 项目类型的不同，模块的根所在的位置也不同。当使用 `--bin` 创建一个可执行的 Rust 项目时，模块的根是 `src/main.rs` 文件；而当使用 `--lib` 创建一个 Rust 库项目时，模块的根是 `src/lib.rs` 文件。在模块的根文件中，我们需要声明所有可能会用到的子模块。如果不声明的话，即使子模块对应的文件存在，Rust 编译器也不会用到它们。如上面的代码片段中，我们就在根文件 `src/main.rs` 中通过 `mod lang_items;` 声明了子模块 `lang_items` ，该子模块实现在文件 `src/lang_item.rs` 中，我们将项目中所有的语义项放在该模块中。

当一个子模块比较复杂的时候，它往往不会被放在一个独立的文件中，而是放在一个 `src` 目录下与子模块同名的子目录之下，在后面的章节中我们常会用到这种方法。

每个模块可能会对其他模块公开一些变量、类型或函数，而该模块的其他内容则是对其他模块不可见的，也即其他模块不允许引用或访问这些内容。在模块内，仅有被显式声明为 `pub` 的内容才会对其他模块公开。Rust 类内部声明的属性域和方法也可以对其他类公开或是不对其他类公开，这取决于它们是否被声明为 `pub` 。我们在 C++/Java 语言中能够找到相同功能的关键字：即 `public/private` 。提供上述可见性机制的原因在于让其他类/模块能够访问当前类/模块公开提供的内容而无需关心它们是如何实现的，它们实际上也无法看到这些具体实现，因为这些具体实现并未向它们公开。编译器会对可见性进行检查，例如，当一个类/模块试图访问其他类/模块未公开的方法时，将无法通过编译。

我们可以使用绝对路径或相对路径来引用其他模块或当前模块的内容。参考上面的 `use core::panic::PanicInfo;` ，类似 C++ ，我们将模块的名字按照层级由浅到深排列，并在相邻层级之间使用分隔符 `::` 进行分隔。路径的最后一级（如 `PanicInfo`）则表示我们具体要引用或访问的内容，可能是变量、类型或者方法名。当通过绝对路径进行引用时，路径最开头可能是项目依赖的一个外部库的名字，或者是 `crate` 表示项目自身的根模块。在后面的章节中，我们会多次用到它们。

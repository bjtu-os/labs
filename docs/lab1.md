# Exercise 1 独立可执行程序
[TOC]



## Part 1 应用执行与平台支持

### 1.1 创建一个简单的Rust程序

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

它是最简单的 Rust 应用

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

如我们预想的一样，我们在屏幕上看到了一行 `Hello, world!` 。

但是，需要注意到**我们所享受到的编程和执行程序的方便性并不是理所当然的，背后有着从硬件到软件的多种机制的支持。**简而言之，我们使用了操作系统和Rust语言本身提供的许多便捷接口。我们在打印 `Hello, world!` 时使用的 `println!` 宏正是由 Rust 标准库 std 和 GNU Libc 库等提供的。

从内核/操作系统的角度看来，它上的上层软件的一切操作都属于用户态，而操作系统自身属于内核态。无论用户态应用如何编写，是手写汇编代码，还是基于某种高级编程语言调用其标准库或三方库，某些功能总要直接或间接的通过内核/操作系统提供的 **系统调用** (System Call) 来实现。

### 1.2 平台

对于一份用某种编程语言实现的应用程序源代码而言，编译器在将其通过编译、链接得到可执行文件的时候需要知道程序要在哪个 **平台** (Platform) 上运行。这里平台主要是指 CPU 类型、操作系统类型和标准运行时库的组合。由于我们要自己开发操作系统，我们打算基于 **RISC-V 架构而非 x86 系列架构**指令来编译我们的程序，原因在于x86指令非常复杂，对于我们编写的操作系统而言没有必要使用如此复杂的指令集架构，而RISC-V相对简单，且它的这些功能已经足以用来构造一个具有相当抽象能力且可以运行的简洁内核了。

我们如果想编译出不同指令集的代码则需要

在os根目录下的.cargo文件夹(没有就创建)新建config文件，并写入：

```
# os/.cargo/config
[build]
target = "riscv64gc-unknown-none-elf"
```

它的含义是：

 CPU 架构是 `riscv64gc`，厂商是 `unknown`，操作系统是 `none`，`elf 表示没有标准的运行时库（表明没有任何系统调用的封装支持）`，但可以生成 ELF 格式的执行程序。

此时如果重新编译代码，编译器就会更换CPU指令集使用riscv64gc

## Part 2 移除标准库依赖

操作系统是很底层的应用软件，它不应该依赖Rust的标准库，因为标准库本身需要操作系统本身支持。

### 2.1 移除println!()宏

`println!` 宏所在的 Rust 标准库 std 需要通过系统调用获得操作系统的服务，而如果要构建运行在裸机上的操作系统，就不能再依赖标准库了。所以我们第一步要尝试移除 `println!` 宏及其所在的标准库。

我们在 `main.rs` 的开头加上一行 `#![no_std]` 来告诉 Rust 编译器不使用 Rust 标准库 std ，转而使用核心库 core（core库不需要操作系统的支持）。**同时记得注释掉之前使用的println!()宏**，重新编译执行，你会得到一个报错的程序，出错信息大概长这样：

```
`#[panic_handler]` function required, but not found
```

在使用 Rust 编写应用程序的时候，我们常常在遇到了一些无法恢复的致命错误（panic），导致程序无法继续向下运行。这时手动或自动调用 `panic!` 宏来打印出错的位置，让我们能够意识到它的存在，并进行一些后续处理。

Rust编译器在编译程序时，从安全性考虑，需要有 `panic!` 宏的具体实现。

在标准库 std 中提供了关于 `panic!` 宏的具体实现，其大致功能是打印出错位置和原因并杀死当前应用。

但是由于我们人工删掉了标准库依赖，导致这个宏失效了，因此我们需要手动实现一下这个宏：

```rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`#[panic_handler]` 是一种编译指导属性，用于标记核心库core中的 `panic!` 宏要对接的函数（该函数实现对致命错误的具体处理）。该编译指导属性所标记的函数需要具有 `fn(&PanicInfo) -> !` 函数签名，函数可通过 `PanicInfo` 数据结构获取致命错误的相关信息。这样Rust编译器就可以把核心库core中的 `panic!` 宏与 `#[panic_handler]` 指向的panic函数实现合并在一起，使得no_std程序具有类似std库的应对致命错误的功能。

再重新编译一下这个源代码：

```rust
#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

fn main() {
    //println!("Hello, world!");
}
```

还是编译出错，好不好玩，意不意外？

这次出错的原因是：

```
error: requires `start` lang_item
```

### 2.2 移除main函数

编译器提醒我们缺少一个名为 `start` 的语义项。

我们回忆一下，Rust语言标准库和三方库作为应用程序的执行环境需要负责在执行应用程序之前进行一些初始化工作，然后才跳转到应用程序的入口点（也就是跳转到我们编写的 `main` 函数）开始执行。

事实上 `start` 语义项代表了标准库 std 在**执行应用程序之前**需要进行的一些**初始化工作**。由于我们禁用了标准库，编译器也就找不到这项功能的实现了。

最简单粗暴的方法就是在 `main.rs` 的开头加入设置 `#![no_main]` 告诉编译器我们没有一般意义上的 `main` 函数，并将原来的 `main` 函数删除，压根不让编译器使用这项功能。在失去了 `main` 函数的情况下，编译器也就不需要完成所谓的初始化工作了。

```rust
#![no_std]
#![no_main]
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

```

此时你再编译这个源代码，就不会报错了。

```
Finished dev [unoptimized + debuginfo] target(s) in 0.24s
```



## Part 3 构建用户态执行环境

### 3.1 执行环境初始化

我们之前把主函数给删除了，那只是权宜之计，我们最终还是需要能让程序跑起来的。因此现在我们需要给Rust编译器提供不依赖标准库的入口函数`_start`

在之前的代码中追加如下代码段：

```rust
#[no_mangle]
extern "C" fn _start() {
    loop{};
}
```

重新编译，使用反汇编工具导出汇编程序。可以看到如下输出：

```bash
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s

[反汇编导出汇编程序]
$ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
   target/riscv64gc-unknown-none-elf/debug/os:       file format elf64-littleriscv

   Disassembly of section .text:

   0000000000011120 <_start>:
   ;     loop {}
     11120: 09 a0            j       2 <_start+0x2>
     11122: 01 a0            j       0 <_start+0x2>
```

可以看到这就是一个死循环程序。但是现在可以执行了。

### 3.2 提供执行退出机制

将原来中的`main.rs`内容替换为：

```rust
#![no_std]
#![no_main]
use core::panic::PanicInfo;


#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

const SYSCALL_EXIT: usize = 93;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id,
        );
    }
    ret
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}

#[no_mangle]
extern "C" fn _start() {
    sys_exit(9);
}
```

我们实现了一个系统调用叫`sys_exit`,整段程序一旦启动就会退出。

我们编译执行一下修改后的程序：

```bash
$ cargo build --target riscv64gc-unknown-none-elf
  Compiling os v0.1.0 (/media/chyyuu/ca8c7ba6-51b7-41fc-8430-e29e31e5328f/thecode/rust/os_kernel_lab/os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.26s

[打印程序的返回值]
$ qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os; echo $?
9
```

可以看到，返回的结果确实是 `9` 。这样，我们勉强完成了一个简陋的用户态最小化执行环境。

### 3.3 重新实现println!宏

#### 0x01 封装`SYSCALL_WRITE`调用

```rust
const SYSCALL_WRITE: usize = 64;

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
  syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

```

#### 0x02 实现 Write Trait 数据结构

注意：*trait* 类似于其他语言中，常被称为 **接口**（*interfaces*）的功能，虽然有一些不同。

实现基于 `Write` Trait 的数据结构，并完成 `Write` Trait 所需要的 `write_str` 函数，并用 `print` 函数进行包装。最后，基于 `print` 函数，实现Rust语言 **格式化宏** ( [formatting macros](https://doc.rust-lang.org/std/fmt/#related-macros) )。

```rust
#![no_std]
#![no_main]
use core::panic::PanicInfo;


#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

const SYSCALL_EXIT: usize = 93;
const SYSCALL_WRITE: usize = 64;


fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id,
        );
    }
    ret
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}


#[no_mangle]
extern "C" fn _start() {
    println!("Hello, world!");
    sys_exit(9);
}

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
  syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        sys_write(1, s.as_bytes());
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();

}
use core::fmt::{self, Write};
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

编译执行，可以发现`Hello world !`被正常打印。


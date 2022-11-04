# 基于 SBI 服务完成输出和关机

## 本节导读

本节我们将进行构建操作系统内核的最后一个步骤，即基于 RustSBI 提供的服务完成在屏幕上打印 `Hello world!` 和关机操作。事实上，作为对我们之前提到的 *应用程序执行环境* 的细化，RustSBI 介于底层硬件和内核之间，是我们内核的底层执行环境。本节将会提到执行环境除了为上层应用进行初始化的第二种职责：即在上层应用运行时提供服务。本节的代码涉及的汇编和 Rust 的细节较多，不必完全理解其含义，重点在于将内核成功运行起来。

## 使用 RustSBI 提供的服务

之前我们对 RustSBI 的了解仅限于它会在计算机启动时进行它所负责的环境初始化工作，并将计算机控制权移交给内核。但实际上作为内核的执行环境，它还有另一项职责：即在内核运行时响应内核的请求为内核提供服务。当内核发出请求时，计算机会转由 RustSBI 控制来响应内核的请求，待请求处理完毕后，计算机控制权会被交还给内核。从内存布局的角度来思考，每一层执行环境（或称软件栈）都对应到内存中的一段代码和数据，这里的控制权转移指的是 CPU 从执行一层软件的代码到执行另一层软件的代码的过程。这个过程和函数调用比较像，但是内核无法通过函数调用来请求 RustSBI 提供的服务，这是因为内核并没有和 RustSBI 链接到一起，我们仅仅使用 RustSBI 构建后的可执行文件，因此内核对于 RustSBI 的符号一无所知。事实上，内核需要通过另一种复杂的方式来 “调用” RustSBI 的服务：

```rust
// os/src/main.rs
mod sbi;

// os/src/sbi.rs
use core::arch::asm;
#[inline(always)]
fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
    let mut ret;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") arg0 => ret,
            in("x11") arg1,
            in("x12") arg2,
            in("x17") which,
        );
    }
    ret
}
```

我们将内核与 RustSBI 通信的相关功能实现在子模块 `sbi` 中，因此我们需要在 `main.rs` 中加入 `mod sbi` 将该子模块加入我们的项目。在 `os/src/sbi.rs` 中，我们首先关注 `sbi_call` 的函数签名， `which` 表示请求 RustSBI 的服务的类型（RustSBI 可以提供多种不同类型的服务）， `arg0` ~ `arg2` 表示传递给 RustSBI 的 3 个参数，而 RustSBI 在将请求处理完毕后，会给内核一个返回值，这个返回值也会被 `sbi_call` 函数返回。尽管我们还不太理解函数 `sbi_call` 的具体实现，但目前我们已经知道如何使用它了：当需要使用 RustSBI 服务的时候调用它就行了。

在 `sbi.rs` 中我们定义 RustSBI 支持的服务类型常量，它们并未被完全用到：

```rust
// os/src/sbi.rs
#![allow(unused)] // 此行请放在该文件最开头
const SBI_SET_TIMER: usize = 0;
const SBI_CONSOLE_PUTCHAR: usize = 1;
const SBI_CONSOLE_GETCHAR: usize = 2;
const SBI_CLEAR_IPI: usize = 3;
const SBI_SEND_IPI: usize = 4;
const SBI_REMOTE_FENCE_I: usize = 5;
const SBI_REMOTE_SFENCE_VMA: usize = 6;
const SBI_REMOTE_SFENCE_VMA_ASID: usize = 7;
const SBI_SHUTDOWN: usize = 8;
```

如字面意思，服务 `SBI_CONSOLE_PUTCHAR` 可以用来在屏幕上输出一个字符。我们将这个功能封装成 `console_putchar` 函数：

```rust
// os/src/sbi.rs
pub fn console_putchar(c: usize) {
    sbi_call(SBI_CONSOLE_PUTCHAR, c, 0, 0);
}
```

注意我们并未使用 `sbi_call` 的返回值，因为它并不重要。类似的，还可以将关机服务 `SBI_SHUTDOWN` 封装成 `shutdown` 函数：

```rust
// os/src/sbi.rs
pub fn shutdown() -> ! {
    sbi_call(SBI_SHUTDOWN, 0, 0, 0);
    panic!("It should shutdown!");
}
```

## 实现格式化输出

`console_putchar` 的功能过于受限，如果想打印一行 `Hello world!` 的话需要进行多次调用。能否像本 Lab 第一节那样使用 `println!` 宏一行就完成输出呢？因此我们尝试自己编写基于 `console_putchar` 的 `println!` 宏。

```rust
// os/src/main.rs
#[macro_use]
mod console;

// os/src/console.rs
use crate::sbi::console_putchar;
use core::fmt::{self, Write};

struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for c in s.chars() {
            console_putchar(c as usize);
        }
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}

#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

我们在 `console` 子模块中编写 `println!` 宏。结构体 `Stdout` 不包含任何字段，因此它被称为类单元结构体（Unit-like structs）。 `core::fmt::Write` trait 包含一个用来实现 `println!` 宏很好用的 `write_fmt` 方法，为此我们准备为结构体 `Stdout` 实现 `Write` trait 。在 `Write` trait 中， `write_str` 方法必须实现，因此我们需要为 `Stdout` 实现这一方法，它并不难实现，只需遍历传入的 `&str` 中的每个字符并调用 `console_putchar` 就能将传入的整个字符串打印到屏幕上。

在此之后 `Stdout` 便可调用 `Write` trait 提供的 `write_fmt` 方法并进而实现 `print` 函数。在声明宏（Declarative macros） `print!` 和 `println!` 中会调用 `print` 函数完成输出。

现在我们可以在 `rust_main` 中使用 `print!` 和 `println!` 宏进行格式化输出了。

## 处理致命错误

错误处理是编程的重要一环，它能够保证程序的可靠性和可用性，使得程序能够从容应对更多突发状况而不至于过早崩溃。不同于 C 的返回错误编号 `errno` 模型和 C++/Java 的 `try-catch` 异常捕获模型，Rust 将错误分为可恢复和不可恢复错误两大类。这里我们主要关心不可恢复错误。和 C++/Java 中一个异常被抛出后始终得不到处理一样，在 Rust 中遇到不可恢复错误，程序会直接报错退出。例如，使用 `panic!` 宏便会直接触发一个不可恢复错误并使程序退出。不过在我们的内核中，目前不可恢复错误的处理机制还不完善：

```rust
// os/src/lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

可以看到，在目前的实现中，当遇到不可恢复错误的时候，被标记为语义项 `#[panic_handler]` 的 `panic` 函数将会被调用，然而其中只是一个死循环，会使得计算机卡在这里。借助前面实现的 `println!` 宏和 `shutdown` 函数，我们可以在 `panic` 函数中打印错误信息并关机：

```rust
// os/src/main.rs
#![feature(panic_info_message)]

// os/src/lang_item.rs
use crate::sbi::shutdown;
use core::panic::PanicInfo;

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    if let Some(location) = info.location() {
        println!(
            "Panicked at {}:{} {}",
            location.file(),
            location.line(),
            info.message().unwrap()
        );
    } else {
        println!("Panicked: {}", info.message().unwrap());
    }
    shutdown()
}
```

我们尝试打印更加详细的信息，包括 panic 所在的源文件和代码行数。我们尝试从传入的 `PanicInfo` 中解析这些信息，如果解析成功的话，就和 panic 的报错信息一起打印出来。我们需要在 `main.rs` 开头加上 `#![feature(panic_info_message)]` 才能通过 `PanicInfo::message` 获取报错信息。当打印完毕之后，我们直接调用 `shutdown` 函数关机。

为了测试我们的实现是否正确，我们将 `rust_main` 改为：

```rust
// os/src/main.rs
#[no_mangle]
pub fn rust_main() -> ! {
    clear_bss();
    println!("Hello, world!");
    panic!("Shutdown machine!");
}
```

在 `os` 主目录下，使用该命令去除内核可执行文件中的元数据，就如同之前章节 **编写操作系统内核** 我们做的一样：

```shell
$ rust-objcopy --strip-all target/riscv64gc-unknown-none-elf/release/os -O binary target/riscv64gc-unknown-none-elf/release/os.bin
```

再编译构建我们现在的代码：

```shell
$ cargo build --release
```

使用 Qemu 运行我们的内核，注意 `path/to/rustsbi-qemu.bin` 改成你自己存储 `rustsbi-qemu.bin` 的路径：

```shell
$ qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios path/to/rustsbi-qemu.bin \
    -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
```

运行结果为：

```
[RustSBI output]
Hello, world!
Panicked at src/main.rs:18 Shutdown machine!
```

可以看到，panic 所在的源文件和代码行数被正确报告，这将为我们后续章节的开发和调试带来很大方便。至此，Lab 2 就已经完成了。
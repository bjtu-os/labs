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
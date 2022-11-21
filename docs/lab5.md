# 分时多任务系统与抢占式调度

## 导读

上一章我们介绍了协作式操作系统，依靠应用程序之间的协作来完成任务调度和切换。本章的重点是操作系统对中断的处理和对应用程序的抢占，并设计实现更加公平和高效交互的抢占式操作系统。为此，我们需要对 **任务** 的概念进行进一步扩展和延伸：

- 分时多任务：操作系统管理每个应用程序，以时间片为单位来分时占用处理器运行应用。
- 时间片轮转调度：操作系统在一个程序用完其时间片后，就抢占当前程序并调用下一个程序执行，周而复始，形成对应用程序在任务级别上的时间片轮转调度。

### 分时多任务系统的背景

上一个 Lab 我们介绍了多道程序，它是一种允许应用在等待外设时主动切换到其他应用来达到总体 CPU 利用率最高的设计。在多道程序盛行的时候，计算机应用的开发者和计算机管理者是两类人，开发者不能管理计算机，对于应用的运行过程没有太多控制权，而计算机管理者的目标是总体 CPU 利用率最高，可以换成一个等价的指标： **吞吐量** (Throughput) 。大概可以理解为在某个时间点将一组应用放进去，要求在一段固定的时间之内执行完毕的应用最多，或者是总进度百分比最大。因此，所有的应用和编写应用的程序员都有这样的共识：只要 CPU 一直在做实际的工作就好。

从现在的眼光来看，当时的应用更多是一种 **后台应用** (Background Application) ，在将它加入执行队列之后我们只需定期确认它的运行状态。而随着技术的发展，涌现了越来越多的 **交互式应用** (Interactive Application) ，它们要达成的一个重要目标就是提高用户（应用的使用者和开发者）操作的响应速度，减少 **延迟** （Latency），这样才能优化应用的使用体验和开发体验。对于这些应用而言，即使需要等待外设或某些事件，它们也不会倾向于主动 yield 交出 CPU 使用权，因为这样可能会带来无法接受的延迟。也就是说，应用之间更多的是互相竞争宝贵的硬件资源，而不是相互合作。

如果应用自己很少 yield ，操作系统内核就要开始收回之前下放的权力，由它自己对 CPU 资源进行集中管理并合理分配给各应用，这就是内核需要提供的任务调度能力。我们可以将多道程序的调度机制分类成 **协作式调度** (Cooperative Scheduling) ，因为它的特征是：只要一个应用不主动 yield 交出 CPU 使用权，它就会一直执行下去。与之相对， **抢占式调度** (Preemptive Scheduling) 则是应用 *随时* 都有被内核切换出去的可能。

现代的任务调度算法基本都是抢占式的，它要求每个应用只能连续执行一段时间，然后内核就会将它强制性切换出去。一般将 **时间片** (Time Slice) 作为应用连续执行时长的度量单位，每个时间片可能在毫秒量级。调度算法需要考虑：每次在换出之前给一个应用多少时间片去执行，以及要换入哪个应用。可以从性能（主要是吞吐量和延迟两个指标）和 **公平性** (Fairness) 两个维度来评价调度算法，后者要求多个应用分到的时间片占比不应差距过大。

### 时间片轮转调度

简单起见，我们使用 **时间片轮转算法** (RR, Round-Robin) 来对应用进行调度，只要对它进行少许拓展就能完全满足我们的需求。本 Lab 中我们仅需要最原始的 RR 算法，用文字描述的话就是维护一个任务队列，每次从队头取出一个应用执行一个时间片，然后把它丢到队尾，再继续从队头取出一个应用，以此类推直到所有的应用执行完毕。

## 代码示例

首先 `clone` 或者更新代码仓库。本 Lab 的代码都放在 `code/lab5` 下面：

```shell
$ cd os

# 构建并运行代码 (Makefile 中已包含了 user 目录下的 make build 命令)
$ make run -j $(nproc)

...
[kernel] Hello, world!
...
power_3 [1730000/2000000]
power_3 [1740000/2000000]
power_3 [1750000/2000000]
power_3 [1760000/2000000]
power_3 [1770000/2000000]
power_3 [1780000/2000000]
power_3 [1790000/2000000]
power_3 [1800000/2000000]
power_3 [1810000/2000000]
power_3 [1820000/2000000]
power_5 [870000/1400000]
power_5 [880000/1400000]
power_5 [890000/1400000]
...
7^160000 = 667897727(MOD 998244353)
Test power_7 OK!
[kernel] Application exited with code 0
Test sleep OK!
[kernel] Application exited with code 0
All applications completed!
```

分时多任务系统应用分为两种。编号为 00/01/02 的应用分别会计算质数 3/5/7 的幂次对一个大质数取模的余数，并会将结果阶段性输出。从运行结果中我们可以看到，三个任务的计算结果交错输出，这说明应用程序在执行过程中确实被抢占了。

## RISC-V 架构中的中断

时间片轮转调度的核心机制就在于计时。操作系统的计时功能是依靠硬件提供的时钟中断来实现的。在介绍时钟中断之前，我们先简单介绍一下中断。

在 RISC-V 架构语境下， **中断** (Interrupt) 和我们 **Lab 3 批处理系统**中介绍的异常（包括程序错误导致或执行 Trap 类指令如用于系统调用的 `ecall` ）一样都是一种 Trap ，但是它们被触发的原因却是不同的。对于某个处理器核而言， 异常与当前 CPU 的指令执行是 **同步** (Synchronous) 的，异常被触发的原因一定能够追溯到某条指令的执行；而中断则 **异步** (Asynchronous) 于当前正在进行的指令，也就是说中断来自于哪个外设以及中断如何触发完全与处理器正在执行的当前指令无关。

> **从底层硬件的角度区分同步和异步**
>
> 从底层硬件的角度可能更容易理解这里所提到的同步和异步。以一个处理器的五级流水线设计而言，里面含有取指、译码、算术、访存、寄存器等单元，都属于执行指令所需的硬件资源。那么假如某条指令的执行出现了问题，一定能被其中某个单元看到并反馈给流水线控制单元，从而它会在执行预定的下一条指令之前先进入异常处理流程。也就是说，异常在这些单元内部即可被发现并解决。
>
> 而对于中断，可以理解为发起中断的是一套与处理器执行指令无关的电路（从时钟中断来看就是简单的计数和比较器），这套电路仅通过一根导线接入处理器。当外设想要触发中断的时候则输入一个高电平或正边沿，处理器会在每执行完一条指令之后检查一下这根线，看情况决定是继续执行接下来的指令还是进入中断处理流程。也就是说，大多数情况下，指令执行的相关硬件单元和可能发起中断的电路是完全独立 **并行** (Parallel) 运行的，它们中间只有一根导线相连。

在不考虑指令集拓展的情况下，RISC-V 架构中定义了如下中断：

| Interrupt | Exception Code | Description                   |
| --------- | -------------- | ----------------------------- |
| 1         | 1              | Supervisor software interrupt |
| 1         | 3              | Machine software interrupt    |
| 1         | 5              | Supervisor timer interrupt    |
| 1         | 7              | Machine timer interrupt       |
| 1         | 9              | Supervisor external interrupt |
| 1         | 11             | Machine external interrupt    |

RISC-V 的中断可以分成三类：

- **软件中断** (Software Interrupt)：由软件控制发出的中断
- **时钟中断** (Timer Interrupt)：由时钟电路发出的中断
- **外部中断** (External Interrupt)：由外设发出的中断

另外，相比于异常，中断和特权级之间的联系更为紧密，可以看到这三种中断每一个都有 M/S 特权级两个版本。中断的特权级可以决定该中断是否会被屏蔽，以及需要 Trap 到 CPU 的哪个特权级进行处理。

在判断中断是否会被屏蔽的时候，有以下规则：

- 如果中断的特权级低于 CPU 当前的特权级，则该中断会被屏蔽，不会被处理；
- 如果中断的特权级高于与 CPU 当前的特权级或相同，则需要通过相应的 CSR 判断该中断是否会被屏蔽。

以内核所在的 S 特权级为例，中断屏蔽相应的 CSR 有 `sstatus` 和 `sie` 。`sstatus` 的 `sie` 为 S 特权级的中断使能，能够同时控制三种中断，如果将其清零则会将它们全部屏蔽。即使 `sstatus.sie` 置 1 ，还要看 `sie` 这个 CSR，它的三个字段 `ssie/stie/seie` 分别控制 S 特权级的软件中断、时钟中断和外部中断的中断使能。比如对于 S 态时钟中断来说，如果 CPU 不高于 S 特权级，需要 `sstatus.sie` 和 `sie.stie` 均为 1 该中断才不会被屏蔽；如果 CPU 当前特权级高于 S 特权级，则该中断一定会被屏蔽。

如果中断没有被屏蔽，那么接下来就需要软件进行处理，而具体到哪个特权级进行处理与一些中断代理 CSR 的设置有关。默认情况下，所有的中断都需要到 M 特权级处理。而通过软件设置这些中断代理 CSR 之后，就可以到低特权级处理，但是 Trap 到的特权级不能低于中断的特权级。事实上所有的中断/异常默认也都是到 M 特权级处理的。

简单的来说就是：

- U 特权级的应用程序发出系统调用或产生错误异常都会跳转到 S 特权级的操作系统内核来处理；
- S 特权级的时钟/软件/外部中断产生后，都会跳转到 S 特权级的操作系统内核来处理。

这里我们还需要对 **Lab 3 批处理系统** 介绍的系统调用和异常发生时的硬件机制做一下与中断相关的补充。默认情况下，当中断产生并进入某个特权级之后，在中断处理的过程中同特权级的中断都会被屏蔽。中断产生后，硬件会完成如下事务：

- 当中断发生时，`sstatus.sie` 字段会被保存在 `sstatus.spie` 字段中，同时把 `sstatus.sie` 字段置零，这样软件在进行后续的中断处理过程中，所有 S 特权级的中断都会被屏蔽；
- 当软件执行中断处理完毕后，会执行 `sret` 指令返回到被中断打断的地方继续执行，硬件会把 `sstatus.sie` 字段恢复为 `sstatus.spie` 字段内的值。

也就是说，如果不去手动设置 `sstatus` CSR ，在只考虑 S 特权级中断的情况下，是不会出现 **嵌套中断** (Nested Interrupt) 的。嵌套中断是指在处理一个中断的过程中再一次触发了中断。由于默认情况下，在软件开始响应中断前， 硬件会自动禁用所有同特权级中断，自然也就不会再次触发中断导致嵌套中断了。

> **嵌套中断与嵌套 Trap**
>
> 嵌套中断可以分为两部分：在处理一个中断的过程中又被同特权级/高特权级中断所打断。默认情况下硬件会避免同特权级再次发生，但高特权级中断则是不可避免的会再次发生。
>
> 嵌套 Trap 则是指处理一个 Trap（可能是中断或异常）的过程中又再次发生 Trap ，嵌套中断是嵌套 Trap 的一个特例。在内核开发时我们需要仔细权衡哪些嵌套 Trap 应当被允许存在，哪些嵌套 Trap 又应该被禁止，这会关系到内核的执行模型。

## 时钟中断与计时器

由于软件（特别是操作系统）需要一种计时机制，RISC-V 架构要求处理器要有一个内置时钟，其频率一般低于 CPU 主频。此外，还有一个计数器用来统计处理器自上电以来经过了多少个内置时钟的时钟周期。在 RISC-V 64 架构上，该计数器保存在一个 64 位的 CSR `mtime` 中，我们无需担心它的溢出问题，在内核运行全程可以认为它是一直递增的。

另外一个 64 位的 CSR `mtimecmp` 的作用是：一旦计数器 `mtime` 的值超过了 `mtimecmp`，就会触发一次时钟中断。这使得我们可以方便的通过设置 `mtimecmp` 的值来决定下一次时钟中断何时触发。

可惜的是，它们都是 M 特权级的 CSR ，而我们的内核处在 S 特权级，是不被允许直接访问它们的。好在运行在 M 特权级的 SEE （这里是RustSBI）已经预留了相应的接口，我们可以调用它们来间接实现计时器的控制：

```rust
// os/src/timer.rs

use riscv::register::time;

pub fn get_time() -> usize {
    time::read()
}
```

`timer` 子模块的 `get_time` 函数可以取得当前 `mtime` 计数器的值；

```rust
// os/src/sbi.rs

const SBI_SET_TIMER: usize = 0;

pub fn set_timer(timer: usize) {
    sbi_call(SBI_SET_TIMER, timer, 0, 0);
}

// os/src/timer.rs

use crate::config::CLOCK_FREQ;
const TICKS_PER_SEC: usize = 100;

pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}
```

- 代码片段第 5 行， `sbi` 子模块有一个 `set_timer` 调用，是一个由 SEE 提供的标准 SBI 接口函数，它可以用来设置 `mtimecmp` 的值。

- 代码片段第 14 行， `timer` 子模块的 `set_next_trigger` 函数对 `set_timer` 进行了封装，它首先读取当前 `mtime` 的值，然后计算出 10ms 之内计数器的增量，再将 `mtimecmp` 设置为二者的和。这样，10ms 之后一个 S 特权级时钟中断就会被触发。

  至于增量的计算方式，常数 `CLOCK_FREQ` 是一个预先获取到的各平台不同的时钟频率，单位为赫兹，也就是一秒钟之内计数器的增量。它可以在 `config` 子模块中找到。`CLOCK_FREQ` 除以常数 `TICKS_PER_SEC` 即是下一次时钟中断的计数器增量值。

后面可能还有一些计时的操作，比如统计一个应用的运行时长，我们再设计一个函数：

```rust
// os/src/timer.rs

const MICRO_PER_SEC: usize = 1_000_000;

pub fn get_time_us() -> usize {
    time::read() / (CLOCK_FREQ / MICRO_PER_SEC)
}
```

`timer` 子模块的 `get_time_us` 以微秒为单位返回当前计数器的值，这让我们终于能对时间有一个具体概念了。实现原理就不再赘述。

新增一个系统调用，方便应用获取当前的时间：

```rust
/// 功能：获取当前的时间，保存在 TimeVal 结构体 ts 中，_tz 在我们的实现中忽略
/// 返回值：返回是否执行成功，成功则返回 0
/// syscall ID：169
fn sys_get_time(ts: *mut TimeVal, _tz: usize) -> isize;

#[repr(C)]
pub struct TimeVal {
    pub sec: usize,
    pub usec: usize,
}
```

它在内核中的实现只需调用 `get_time_us` 函数即可。

## 抢占式调度

有了时钟中断和计时器，抢占式调度就很容易实现了：

```rust
// os/src/trap/mod.rs

match scause.cause() {
    Trap::Interrupt(Interrupt::SupervisorTimer) => {
        set_next_trigger();
        suspend_current_and_run_next();
    }
}
```

我们只需在 `trap_handler` 函数下新增一个条件分支跳转，当发现触发了一个 S 特权级时钟中断的时候，首先重新设置一个 10ms 的计时器，然后调用上一小节提到的 `suspend_current_and_run_next` 函数暂停当前应用并切换到下一个。

为了避免 S 特权级时钟中断被屏蔽，我们需要在执行第一个应用之前进行一些初始化设置：

```rust
// os/src/main.rs

#[no_mangle]
pub fn rust_main() -> ! {
    clear_bss();
    println!("[kernel] Hello, world!");
    trap::init();
    loader::load_apps();
    trap::enable_timer_interrupt();
    timer::set_next_trigger();
    task::run_first_task();
    panic!("Unreachable in rust_main!");
}

// os/src/trap/mod.rs

use riscv::register::sie;

pub fn enable_timer_interrupt() {
    unsafe { sie::set_stimer(); }
}
```

- 第 9 行设置了 `sie.stie` 使得 S 特权级时钟中断不会被屏蔽；
- 第 10 行则是设置第一个 10ms 的计时器。

这样，当一个应用运行了 10ms 之后，一个 S 特权级时钟中断就会被触发。由于应用运行在 U 特权级，且 `sie` 寄存器被正确设置，该中断不会被屏蔽，而是跳转到 S 特权级内的我们的 `trap_handler` 里面进行处理，并顺利切换到下一个应用。这便是我们所期望的抢占式调度机制。从应用运行的结果也可以看出，三个 `power` 系列应用并没有进行 yield ，而是由内核负责公平分配它们执行的时间片。

有同学可能会注意到，我们并没有将应用初始 Trap 上下文中的 `sstatus` 中的 `SPIE` 位置为 1 。这将意味着 CPU 在用户态执行应用的时候 `sstatus` 的 `SIE` 为 0 ，根据定义来说，此时的 CPU 会屏蔽 S 态所有中断，自然也包括 S 特权级时钟中断。但是可以观察到我们的应用在用尽一个时间片之后能够正常被打断。这是因为当 CPU 在 U 态接收到一个 S 态时钟中断时会被抢占，这时无论 `SIE` 位是否被设置都会进入 Trap 处理流程。

目前在等待某些事件的时候仍然需要 yield ，其中一个原因是为了节约 CPU 计算资源，另一个原因是当事件依赖于其他的应用的时候，由于只有一个 CPU ，当前应用的等待可能永远不会结束。这种情况下需要先将它切换出去，使得其他的应用到达它所期待的状态并满足事件的生成条件，再切换回来。

这里我们先通过 yield 来优化 **轮询** (Busy Loop) 过程带来的 CPU 资源浪费。在 `03sleep` 这个应用中：

```rust
// user/src/bin/03sleep.rs

#[no_mangle]
fn main() -> i32 {
    let current_timer = get_time();
    let wait_for = current_timer + 3000;
    while get_time() < wait_for {
        yield_();
    }
    println!("Test sleep OK!");
    0
}
```

它的功能是等待 3000ms 然后退出。可以看出，我们会在循环里面 `yield_` 来主动交出 CPU 而不是无意义的忙等。其实，现在的抢占式调度会在它循环 10ms 之后切换到其他应用，这样能让内核给其他应用分配更多的 CPU 资源并让它们更早运行结束。

我们的抢占式操作系统就算是实现完毕了。它支持把多个应用的代码和数据放置到内存中；并能够执行每个应用；在应用程序发出 `sys_yield` 系统调用时，协作式地切换应用；并能通过时钟中断来实现抢占式调度并强行切换应用，从而提高了应用执行的灵活性、公平性和交互效率。

> **内核代码执行是否会被中断打断？**
>
> 目前为了简单起见，我们的内核不会被 S 特权级中断所打断，这是因为 CPU 在 S 特权级时， `sstatus.sie` 总为 0 。但这会造成内核对部分中断的响应不及时，因此一种较为合理的做法是允许内核在处理系统调用的时候被打断优先处理某些中断，这是一种允许 Trap 嵌套的设计。
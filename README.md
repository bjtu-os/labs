# BJTU 操作系统课程实验

## 实验概览

本实验旨在一步一步展示如何从零开始用Rust 语言写一个基于 RISC-V 架构的类 Unix 内核 。实验内容主要分为以下几个方面。

- [Lab 0](docs/lab0.md) - 配置操作系统开发的基本环境。
- [Lab 1](docs/lab1.md) - 构建一个独立的不依赖于Rust标准库的可执行程序。
- Lab 2 - 实现裸机上的执行环境以及一个最小化的操作系统内核。
- Lab 3 - 实现一个简单的批处理操作系统并理解特权级的概念。
- Lab 4 - 实现一个支持多道程序和协作式调度的操作系统。
- Lab 5 - 实现一个分时多任务和抢占式调度的操作系统。
- Lab 6 - 实现内存的动态申请和释放管理。
- Lab 7 - 实现进程及进程的管理。

共计8个实验项目，通过实验的方式深入研讨操作系统底层的工作原理，并使用Rust语言逐步实现一个基本的操作系统内核。

## 实验环境要求

1. OS环境配置：对于 Windows10/11 和 macOS 上的用户，可以使用WSL2、VMware Workstation 或 VirtualBox 等相关软件，通过虚拟机方式安装 Ubuntu 22.04 / 20.04，并在上面进行实验
2. Rust环境配置：需要在虚拟机上配置好Rust开发环境。
3. Qemu 模拟器安装：我们需要使用 Qemu 7.0.0 以上版本进行实验，为此，从源码手动编译安装 Qemu 模拟器。

## 前置知识

1. 熟悉操作系统相关知识。比如，一个程序运行中操作系统到底起了什么作用？程序运行中操作系统的内核态和用户态工作情况？OS的系统调用？进程？内存？IO？等。
2. 对计算机体系结构有一定简单了解。比如，裸机是什么？裸机上能跑程序吗？什么是指令集？计算机体系结构？CPU、内存、IO等这些硬件之间如何协调完成计算任务的？
3. 能够读懂简单的汇编语言、能够读懂简单的makefile文件。
4. 熟悉Rust 基本语法，包括 Cargo 项目结构、Trait、函数式编程、Unsafe Rust、错误处理等。[Rust入门教程](docs/rust-tutorial.md)

## 参考资料

1. [函数 - Rust 程序设计语言 中文版 (rustwiki.org.cn)](https://www.rustwiki.org.cn/zh-CN/book/ch03-03-how-functions-work.html)
2. [OS Tutorial Summer of Code 2020：Rust系统编程入门指导](https://github.com/rcore-os/rCore/wiki/os-tutorial-summer-of-code#step-0-自学rust编程大约7天)
3. [Stanford 新开的一门很值得学习的 Rust 入门课程](https://reberhardt.com/cs110l/spring-2020/)
4. [一份简单的 Rust 入门介绍](https://zhuanlan.zhihu.com/p/298648575)
5. [《RustOS Guide》中的 Rust 介绍部分](https://simonkorl.gitbook.io/r-z-rustos-guide/dai-ma-zhi-qian/ex1)
6. [一份简单的Rust宏编程新手指南](http://blog.hubwiz.com/2020/01/30/rust-macro/)
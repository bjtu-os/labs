# Rust 入门教程

## 安装 Rust

### `rustup`：Rust 安装器和版本管理工具

安装 Rust 的主要方式是通过 `rustup` 这一工具，它既是一个 Rust 安装器又是一个版本管理工具。

该教程是基于 macOS、Linux 或其它类 Unix 系统。要下载 `rustup` 并安装 Rust，请在终端中运行以下命令，然后遵循屏幕上的指示。如果你使用的是 Windows 系统，请参见 [“其他安装方式”](https://forge.rust-lang.org/infra/other-installation-methods.html)。

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### `cargo`：Rust 的构建工具和包管理器

你在安装 `rustup` 时，也会安装 Rust 构建工具和包管理器的最新稳定版，即 `cargo`。`cargo` 可以做很多事情：

- `cargo build` 可以构建项目
- `cargo run` 可以运行项目
- `cargo test` 可以测试项目
- `cargo doc` 可以为项目构建文档
- `cargo publish` 可以将库发布到 [crates.io](https://crates.io/)。

要检查你是否安装了 Rust 和 `cargo`，可以在终端中运行：

```shell
cargo --version
```

## 创建新项目

我们将在新的 Rust 开发环境中编写一个小应用。首先用 `cargo` 创建一个新项目。在你的终端中执行：

```shell
cargo new hello-rust
```

这会生成一个名为 `hello-rust` 的新目录，其中包含以下文件：

```
hello-rust
|- Cargo.toml
|- src
  |- main.rs
```

`Cargo.toml` 为 Rust 的清单文件。其中包含了项目的元数据和依赖库。

`src/main.rs` 为编写应用代码的地方。

------

`cargo new` 会生成一个新的 “Hello, world!” 项目！我们可以进入新创建的目录中，执行下面的命令来运行此程序：

```shell
cargo run
```

您应该会在终端中看到如下内容：

```shell
$ cargo run
   Compiling hello-rust v0.1.0 (/Users/ag_dubs/rust/hello-rust)
    Finished dev [unoptimized + debuginfo] target(s) in 1.34s
     Running `target/debug/hello-rust`
Hello, world!
```

## 更改 `cargo` 镜像

因为 `cargo` 默认的下载源在国外，速度比较慢，所以我们可以设置 [清华TUNA](https://mirrors.tuna.tsinghua.edu.cn/help/crates.io-index.git/) 的镜像来加快下载速度。编辑 `cargo` 的配置文件：

```shell
vim ~/.cargo/config
```

添加以下内容：

```toml
[source.crates-io]
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```

## 添加依赖

现在我们来为应用添加依赖。你可以在 [crates.io](https://crates.io/)，即 Rust 包的仓库中找到所有类别的库。在 Rust 中，我们通常把包称作`crates`。

在本项目中，我们使用了名为 [`ferris-says`](https://crates.io/crates/ferris-says) 的库。

我们在 `Cargo.toml` 文件中添加以下信息（从 crate 页面上获取）：

```toml
[dependencies]
ferris-says = "0.2"
```

接着运行：

```shell
cargo build
```

…之后 `cargo` 就会安装该依赖。

运行此命令会创建一个新文件 `Cargo.lock`，该文件记录了本地所用依赖库的精确版本。

要使用该依赖库，我们可以打开 `main.rs`，删除其中所有的内容（它不过是个示例而已），然后在其中添加下面这行代码：

```rust
use ferris_says::say;
```

这样我们就可以使用 `ferris-says` crate 中导出的 `say` 函数了。

## 一个 Rust 小应用

现在我们用新的依赖库编写一个小应用。在 `main.rs` 中添加以下代码：

```rust
use ferris_says::say; // from the previous step
use std::io::{stdout, BufWriter};

fn main() {
    let stdout = stdout();
    let message = String::from("Hello fellow Rustaceans!");
    let width = message.chars().count();

    let mut writer = BufWriter::new(stdout.lock());
    say(message.as_bytes(), width, &mut writer).unwrap();
}
    
```

保存完毕后，我们可以输入以下命令来运行此应用：

```shell
cargo run
```

如果一切正确，您会看到该应用将以下内容打印到了屏幕上：

```
----------------------------
< Hello fellow Rustaceans! >
----------------------------
              \
               \
                 _~^~^~_
             \) /  o o  \ (/
               '_   -   _'
               / '-----' \
    
```

## Learn more!

现在你已经了解了 Rust 基本的构建、运行、包管理功能，接下来你可以在 [官网](https://www.rust-lang.org/zh-CN/learn) 学习更多 Rust 知识！

### 参考

1. Rust 官网入门教程：https://www.rust-lang.org/learn/get-started

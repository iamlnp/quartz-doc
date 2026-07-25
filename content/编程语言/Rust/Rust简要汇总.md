---
状态:
  - 进行中
年份:
  - "2025"
imageNameKey: Rust简要调研
tags:
  - 技术学习
  - rust
  - 编程语言
---

Rust工具： https://www.rust-lang.org/tools/install

# 1 cargo
```rust
cargo new my_test
```

开始于单元包的根节点：在编译一个单元包时，编译器会从单元包的根节点文件开始编译（通常是库单元包中的src/lib.rs，或二进制单元包中的src/main.rs）​。

# 2 rustc

## 2.1 问题

1. rustc版本不匹配
```bash
error: rustc 1.92.0 is not supported by the following packages:
rustfs@0.0.5 requires rustc 1.93.0
rustfs@0.0.5 requires rustc 1.93.0
rustfs@0.0.5 requires rustc 1.93.0
```
**解决方案**：
```bash
# 1. 安装 1.93.0 版本的 Rust（rustup 自动处理编译器和工具链）
rustup install 1.93.0

# 2. 可选：设置为全局默认版本（影响所有 Rust 项目）
rustup default 1.93.0

# 3. 推荐：仅为当前项目设置版本（不影响全局，更安全）
# 进入项目根目录执行
rustup override set 1.93.0

# 4. 验证版本是否切换成功
rustc --version
# 正确输出：rustc 1.93.0 (xxxx 2025-xx-xx)
```
# 3 thread
在 Rust 中，`handle.join().unwrap()` 是用于等待线程完成并获取其返回值的常见操作。
`join()` 方法返回一个 `Result<T, Box<dyn Error>>`，其中 `T` 是被等待线程的返回值类型。使用 `unwrap()` 是一种简单的错误处理方式，它会：
- 如果结果是 `Ok(t)`，则返回内部的值 `t`
- 如果结果是 `Err(e)`，则会触发 panic 并显示错误信息

```rust
fn main() {
    // 创建一个线程并获取其句柄
    let handle = thread::spawn(|| {
        thread::sleep(Duration::from_secs(1));
        "线程执行完成" // 线程的返回值
    });

    // 等待线程完成并获取返回值
    let result = handle.join().unwrap();
    println!("{}", result); // 输出: 线程执行完成
}
```
在 Rust 中，`let _ = handle.join();` 是一种处理线程 JoinHandle 的方式，它的作用是：
1. 调用 `join()` 方法阻塞当前线程，等待被 spawn 的线程执行完成
2. 使用 `let _ =` 忽略 `join()` 返回的 Result 值
与 `handle.join().unwrap()` 不同，这种写法会静默忽略任何可能的错误，包括线程恐慌。

# 4 安装
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
或者
```bash
wget --https-only --secure-protocol=TLSv1_2 -qO- https://sh.rustup.rs | sh
```

刷新环境变量
安装完成后，需要让终端识别新安装的 `rustup` 命令，执行：
```bash
source $HOME/.cargo/env
```

## 4.1 问题

1. 安装rustup时报错：
```bash
[22:35:07] root@ceph-221:/home/code/eza# curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
info: downloading installer
warn: It looks like you have an existing installation of Rust at:
warn: /usr/bin
warn: It is recommended that rustup be the primary Rust installation.
warn: Otherwise you may have confusion unless you are careful with your PATH.
warn: If you are sure that you want both rustup and your already installed Rust
warn: then please reply `y' or `yes' or set RUSTUP_INIT_SKIP_PATH_CHECK to yes
warn: or pass `-y' to ignore all ignorable checks.
error: cannot install while Rust is installed
```
解决方法：
这个错误是因为系统中已经通过包管理器（如 `apt`、`yum` 等）安装了 Rust，而 `rustup` 检测到了现有安装，为了避免环境冲突而终止了安装。解决方法如下：
为了让 `rustup` 成为主要的 Rust 工具链管理器，建议先卸载系统预装的 Rust：
```bash
sudo dnf remove rust cargo
```
卸载完成后，重新运行 `rustup` 安装脚本。

# 5 格式化输出
在 Rust 中，`println!("{:?}", other);` 是一个用于打印变量 `other` 调试信息的宏调用，其中 `{:?}` 是格式化占位符，对应 **`Debug` 格式化输出**。
### 5.1.1 核心作用

- `{:?}` 要求被打印的类型实现了标准库的 `std::fmt::Debug` trait，该 trait 用于提供类型的调试友好格式（通常包含详细的内部结构）。
- 与 `{}`（对应 `Display` trait）不同，`Debug` 输出更偏向开发者调试，格式可能更冗长（例如包含字段名、引号等），且通常由编译器自动派生（通过 `#[derive(Debug)]`），无需手动实现。
- `{:?}` 的变体 `{:#?}` 会生成**带缩进的多行格式**，适合复杂结构（如嵌套的结构体、长列表）：

# 6 所有权
在Rust中，由Copy trait来区分值语义和引用语义。与此同时，Rust也引入了新的语义：复制（Copy）语义和移动（Move）语义。复制语义对应值语义，移动语义对应引用语义。


Rust的借用检查器(borrow checker)，借用检查器会检查所有数据访问是否合法。借用检查依赖于3个紧密关联的概念：所有权、生命周期和借用。
- 所有权(ownership)是一个引申而来的比喻，在Rust中，所有权与针对不再需要的值的清理有关，所有权的概念是相当有限的：一个所有者在它的值的生命周期结束时将被清理。
    - 所有权转移方式
        1. 通过赋值来转移所有权。
        2. 通过函数传递数据，将其作为参数或返回值
- 值的生命周期是一个时间段，在此时间段内对该值的访问是有效的行为。
- 借用一个值意味着要访问它。

所有权的特点：
1. 控制资源（不仅仅是内存）的释放
2. 出借所有权，包括不可变（共享）和可变（独占）的
    1. 通过使用&操作完成所有权租借
    2. 在不可变借用期间，所有者不能修改资源，也不能在进行可变借用
    3. 在可变借用期间，所有者不能访问资源，并且也不能再出借所有权
    4. 不可变借用可以出借多次，因为他不能修改内存数据；可变借用只能出借一次，否则难以预料数据何时何地被修改。
    5. 只读访问的情况，就使用&T；对于读/写访问的情况，就使用&mut T
3. 转移所有权

生命周期参数的目的是帮助借用检查器验证合法的引用，消除悬垂指针

- 借用的生命周期不能长于出借方的生命周期
- 结构体实例的生命周期应短于或等于任意一个成员的生命周期

省略生命周期参数

1. 每个输入位置上省略的生命周期都将成为一个不同的生命周期参数
2. 如果只有一个输入生命周期的位置（不管是否忽略），则该生命周期都将分配给输出生命周期
3. 如果存在多个输入生命周期的位置，但是其中包含着&self或&mut self，则self的生命周期都将分配给输出生命周期

对于Box＜T＞类型来说，如果包含的类型T属于复制语义，则执行按位复制；如果属于移动语义，则移动所有权

 
1. 在Rust中，基本类型是有特殊行为的：它们实现了Copy trait。==基本类型具有复制语义(copy semantics)，而对应地，其他类型就都具有移动语义(move semantics)==。
2. 各种类型在被复制时，都会采取两种可能的模式中的一种：克隆和复制。每种模式都是由一个trait来提供的。克隆是由std::clone::Clone定义的，而复制是由std::marker::Copy定义的
    1. 克隆：
        - 可能是速度慢并且昂贵的
        - 永远不会隐式的发生，总要显式调用.clone()方法
        - 在具体行为上可能有差别，软件包的作者会为包中类型定义出克隆的具体含义
    2. 复制
        - 总是快速并且廉价的
        - 总是隐式的发生
        - 行为上总是相同的，总是安位复制原本的值
# 7 类型

## 7.1 ?Sized

在 Rust 中，`?Sized` 是一个用于 trait bound 的特殊标记，用于表示“允许类型不实现 `Sized` trait”。要理解它，首先需要了解 `Sized` trait 本身：

### 7.1.1 Sized trait 是什么？
`Sized` 是 Rust 中的一个**自动实现的 trait**，用于标记“在编译时已知大小的类型”（例如 `i32`、`String`、自定义结构体等）。  
- 对于这类类型，编译器知道它们在内存中占据的精确大小，因此可以直接在栈上分配，也能作为函数参数/返回值直接传递。  
- 反之，**动态大小类型（DST，Dynamically Sized Type）** 则不实现 `Sized`，例如：
  - 切片 `[T]`（长度未知，需通过 `&[T]` 等指针间接使用）；
  - trait 对象（如 `dyn Trait`，具体类型大小未知）；
  - 字符串字面量的底层类型 `str`（长度未知，需通过 `&str` 使用）。

**`?Sized` 的作用：放宽 `Sized` 限制**
Rust 中，**泛型默认隐含 `Sized` 约束**。例如：
```rust
fn foo<T>(x: T) { ... }
// 等价于
fn foo<T: Sized>(x: T) { ... }
```
这意味着泛型 `T` 只能接受编译时大小已知的类型（`Sized` 类型）。

而 `?Sized` 的作用是**取消这种默认约束**，允许泛型接受“可能不实现 `Sized` 的类型”。例如：
```rust
fn bar<T: ?Sized>(x: &T) { ... }
```
这里 `T` 可以是 `Sized` 类型（如 `i32`），也可以是动态大小类型（如 `str`、`dyn Trait`）。
### 7.1.2 使用场景
`?Sized` 通常用于需要处理动态大小类型的场景，常见情况：
- **接受 trait 对象**：  
  trait 对象（`dyn Trait`）是 DST，因此泛型需要 `?Sized` 才能接受它：
  ```rust
  trait MyTrait { fn do_something(&self); }
  
  // 允许 T 为 dyn MyTrait（DST）
  fn call_trait<T: MyTrait + ?Sized>(x: &T) {
      x.do_something();
  }
  
  // 使用：可以传入任何实现 MyTrait 的类型的引用，或 trait 对象
  let obj: &dyn MyTrait = &SomeType;
  call_trait(obj); // 合法
  ```

- **处理切片或字符串**：  
  直接使用 `[T]` 或 `str` 时（通常通过引用）：
  ```rust
  // 接受 str（DST）的引用
  fn print_str<T: ?Sized>(s: &T) where T: AsRef<str> {
      println!("{}", s.as_ref());
  }
  
  print_str("hello"); // 字符串字面量是 &str，底层是 str（DST）
  ```

- **定义容纳 DST 的类型**：  
  例如自定义智能指针时，指向 DST：
  ```rust
  struct MyBox<T: ?Sized>(*const T);
  
  impl<T: ?Sized> MyBox<T> {
      fn new(x: &T) -> Self {
          MyBox(x as *const T)
      }
  }
  ```

### 7.1.3 注意点
- `?Sized` 仅用于泛型约束，不能直接修饰具体类型。
- 由于 DST 无法在栈上直接存储或作为值传递，使用 `?Sized` 的泛型通常需要通过**引用（`&T`）** 或**指针（如 `Box<T>`、`Rc<T>`）** 间接操作。
- `?Sized` 是“允许不 `Sized`”，而非“必须不 `Sized`”，因此仍能接受 `Sized` 类型。

# 8 log
`RUST_LOG` 是 Rust 生态中用于控制 **日志输出** 的环境变量，主要配合 Rust 的日志库（如 `log`、`tracing`）使用，用于动态调整日志的 **级别**、**模块范围** 和 **输出内容**，无需修改代码即可灵活控制程序的日志行为。
log是Rust 生态中最基础、应用最广泛的 **日志抽象库**（crate），它本身不直接实现日志的输出功能，而是定义了一套统一的日志接口（如日志级别、宏定义），让其他库或应用可以基于这套接口实现日志记录，同时保证不同日志实现之间的兼容性
1. **提供统一的日志接口**：
    定义了 `trace!`、`debug!`、`info!`、`warn!`、`error!` 等日志宏，以及 `Log`、`Level`、`Metadata` 等核心 trait 和枚举，让开发者可以用一致的方式编写日志代码，无需关心底层如何输出（如打印到终端、写入文件、发送到日志服务器等）。
2. **解耦日志生产与消费**：
    库开发者只需依赖 `log` 库编写日志（如 `info!("初始化完成")`），而应用开发者可以自由选择日志的实现方式（如 `env_logger`、`tracing`、`fern` 等），两者通过 `log` 的接口对接，避免了库与特定日志实现的强耦合。
3. 常用搭配的日志实现库
    `log` 库本身不输出日志，必须配合具体的 “日志实现库” 才能生效，常见的有：
    - **`env_logger`**：通过 `RUST_LOG` 环境变量控制日志输出，适合命令行工具和开发调试。
    - **`tracing`**：更强大的日志和追踪库，支持结构化日志、跨度（span）追踪，适合复杂应用和分布式系统。
    - **`fern`**：支持将日志输出到文件、终端等多种目标，可自定义格式和滚动策略。
    - **`simple_logger`**：简单轻量的实现，适合快速上手，无需复杂配置。
## 8.1 日志级别（从低到高）
Rust 日志库定义了 5 个标准级别（级别越高，输出日志越少）：
- `trace`：最详细的调试信息（如函数调用参数、循环步骤），通常用于开发阶段细粒度调试。
- `debug`：调试信息（如关键流程节点、变量值），适合开发和测试环境。
- `info`：普通运行信息（如程序启动、任务完成），生产环境常用。
- `warn`：警告信息（如不影响运行的异常情况，如“配置项缺失，使用默认值”）。
- `error`：错误信息（如功能失败、资源不可用），必须关注的问题。

**规则**：设置某一级别后，会输出该级别及所有更高级别的日志。例如，`RUST_LOG=info` 会输出 `info`、`warn`、`error` 级别的日志。

## 8.2 基本设置方法
### 8.2.1 全局设置日志级别
通过 `RUST_LOG=<级别>` 控制全局日志输出：
```bash
# 只输出 error 及以上级别日志（最简洁）
RUST_LOG=error cargo run

# 输出 info 及以上级别（info, warn, error）
RUST_LOG=info ./my_rust_program

# 输出 debug 及以上级别（开发调试常用）
RUST_LOG=debug cargo test

# 输出所有级别（包括 trace，最详细）
RUST_LOG=trace ./my_rust_program
```

### 8.2.2 限定模块/ crate 的日志范围
通过 `RUST_LOG=<模块路径>=<级别>` 只输出特定模块的日志，避免全局日志冗余：
```bash
# 只输出 my_project 中 network 模块的 debug 级别日志
RUST_LOG=my_project::network=debug cargo run

# 输出 tokio 库的 info 日志 + 自己代码的 debug 日志
RUST_LOG=tokio=info,my_project=debug ./my_program

# 禁用某个模块的日志（设置为 off）
RUST_LOG=my_project::legacy=off ./my_program
```
- 模块路径对应代码中的 `mod` 结构（如 `crate::utils::file`）。
- 可以指定第三方 crate 的名称（如 `tokio`、`hyper`），控制其日志输出。


### 8.2.3 组合设置（多模块 + 不同级别）
用逗号分隔多个规则，实现精细化控制：
```bash
# 全局 info 级别，但 network 模块用 debug，tokio 库用 warn
RUST_LOG=info,my_project::network=debug,tokio=warn ./my_program
```


### 8.2.4 在代码中读取 `RUST_LOG`
需配合日志库（如 `log` + `env_logger`）在程序中初始化日志系统，才能让 `RUST_LOG` 生效。示例：

1. 在 `Cargo.toml` 中添加依赖：
   ```toml
   [dependencies]
   log = "0.4"          # 日志基础库
   env_logger = "0.9"   # 解析 RUST_LOG 的库
   ```

2. 在代码中初始化：
   ```rust
   use log::{info, debug, error};

   fn main() {
       // 初始化日志系统，读取 RUST_LOG 环境变量
       env_logger::init();

       info!("程序启动");
       debug!("配置文件路径: ./config.toml");  // 仅 RUST_LOG>=debug 时输出
       error!("数据库连接失败");
   }
   ```

# 9 参考
1. 《Rust实战》-蒂姆·麦克纳马拉

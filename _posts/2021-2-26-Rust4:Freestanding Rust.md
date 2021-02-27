---
layout: "post"
title: Rust（四）Freestanding/Baremetal Rust
author: LJR
category: 编程语言
tags:
    - rust
---

写操作系统内核的时候，我们的代码无法依赖于任何操作系统的特性，除非是那些自己实现了的。这意味着我们无法使用线程、文件、堆内存、网络、随机数、标准输出等等等。

我们无法使用Rust的标准库，但是仍然可以使用Rust提供的迭代器、闭包、模式匹配、`Option, Result`、格式化字符串([string formatting](https://doc.rust-lang.org/core/macro.write.html))以及所有权系统。

为了使用Rust创建操作系统内核，我们需要创建不依赖于底层操作系统的二进制可执行文件。这样的可执行文件通常被称为"freestanding"或"bare-metal"可执行文件。

## 1. 创建bare-metal可执行文件

### 1.1. 禁用标准库

```rust
#[!no_std]
fn main() {}
// > cargo build
// error: `#[panic_handler]` function required, but not found
// error: language item required, but not found: `eh_personality`
```

### 1.2. 实现panic处理函数

默认情况下，当rust程序出现*panic*时，编译器会调用标准库提供的panic handler function，但是在`no_std`环境下，我们需要自己定义该函数

```rust
use core::panic::PanicInfo;
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`PanicInfo`参数包含了panic发生的文件和行以及可选的panic信息。同时定义该函数为发散函数(*diverging function*)。

### 1.3. 语言项`eh_personality`

语言项(*language items*)是被编译器内部需要的特殊函数或类型。例如，`Copy`trait就是一个语言项，它告知编译器哪些类型有copy语义。在`Copy`的实现中，可以看到`#[lang = "copy"]`属性将它定义成一个语言项。

```rust
#[lang = "copy"]
pub trait Copy : Clone {
    // Empty.
}
```

通常避免自己实现语言项。

`eh_personality`语言项标记了用于实现栈展开(*stack unwinding*)的函数。默认情况下，rust在出现panic的情况下使用该机制来运行栈内变量的析构函数。然而，栈展开需要使用到一些OS具体的库。

#### 1.3.1. 禁用栈展开

当不使用栈展开时，rust也提供了**abort on panic**的选择。这种方式禁止了展开符号(*unwinding symbol*)相关信息的生成因此能够减小二进制文件的大小。最容易的方式是在`Cargo.toml`文件中：

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

如上，同时为`dev`（`cargo build`）和`release`（`cargo build --release`）的panic情况使用了`abort`策略。

接着再次编译的时候会出现

```shell
> cargo build
error: requires `start` lang_item
```

### 1.4. `start`属性

典型的rust二进制程序的执行从C运行时库`crt0`("C runtime zero")开始。它会为c应用程序创建栈、放置命令行参数。接着调用Rust程序的运行时进入点，该进入点由`start`语言项标注。rust仅有一个非常小的运行时，在其中rust会设置栈溢出的guards或者在panic时打印backtrace。最终该运行时调用`main`函数。

freestanding的可执行文件无法访问rust运行时和`crt0`，因而我们需要覆写`crt0`进入点。

#### 1.4.1. 覆写进入点(entry point)

为告知rust编译器我们不需要使用通常的进入点链，需要加上`#![no_main]`，同时移除`main`函数

```rust
#![no_std]
#![no_main]
use core::panic::PanicInfo;
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

使用`#[no_mangle]`属性意味着我们禁用*name mangling*以确保Rust编译器输出的名称为`_start`的函数。若没有这个属性，编译器会生成某个加密的函数名符号。这里使用这个属性，是为了下一步告知linker进入点函数名。

使用`extern "C"`告知编译器使用C调用惯例。函数名使用`_start`是因为大多数系统的默认进入点名为`_start`。

注意到这也是一个发散函数。接着运行`cargo build`，会收到一个链接器错误。

### 1.5. 链接器错误

抛出错的原因在于默认情况下linker会假设程序依赖c运行时。

为了解决这个错误，需要告知linker不需要包含c运行时。

#### 1.5.1. 方法一：构建bare metal目标

默认情况下，rust会尝试为当前host系统环境构建可执行文件。例如，在x86_64 Windows下，rust会尝试构建`.exe`。

为了描述不同的环境，rust使用了被称为*target triple*的字符串。通过运行`rustc --version --verbose`可以查看host系统的*tartget triple*

```
rustc 1.37.0-nightly (0af8e872e 2019-06-30)
binary: rustc
commit-hash: 0af8e872ea5ac77effa59f8d3f8794f12cb8865c
commit-date: 2019-06-30
host: x86_64-unknown-linux-gnu
release: 1.37.0-nightly
LLVM version: 8.0
```

可以注意到*target triple*为`x86_64-unknown-linux-gnu`，分别表示CPU架构(`x86_64`)，发行商(`unknown`)，操作系统(`linux`)以及ABI(`gnu`)。因此当为了host编译时，rust编译器会假设存在默认使用c运行时库的底层linux系统，从而导致linker错误。为了解决linker错误，可以为了另一个不存在底层操作系统的目标而编译。

一个bare metal环境的示例是`thumbv7em-none-eabihf`，其描述的是嵌入式arm的系统。`none`表示无底层操作系统。为了能编译该目标，我们需要将他加入rustup:

```shell
rustup target add thumbv7em-none-eabihf
```

该操作会下载对应系统标准(standart)和核心(core)库。之后即可为该目标编译

```shell
cargo build --target thumbv7em-none-eabihf
```

通过`--target`参数我们为bare metal目标系统交叉编译了可执行文件。

#### 1.5.2. 方法二：使用linker参数

不同的host系统的linker使用不同的参数，下面主要讨论如何解决linux下linker的错误。首先linux下linker的错误如下：

```shell
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x19): undefined reference to `__libc_csu_init'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x25): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status
```

问题在于linker默认包含了c运行时的启动例程，也为`_start`。c运行时的`_start`需要使用标准库的很多符号，然而由于我们使用了`no_std`属性，导致linker无法决议这些符号。为了解决该问题，我们可以使用`-nostartfiles`告知linker它不需要link c启动例程。

一种传递linker参数的方式是通过cargo`cargo rustc`命令，该命令和`cargo build`等价，但同时允许给底层的rust编译器`rustc`传递选项。`rustc`存在`-C link-arg`flag，能够给linker传递参数。总结起来就是

```rust
cargo rustc -- -C link-arg=-nostartfiles
```

我们不需要指明进入点函数的名字，因为linker默认查找名字为`_start`的函数作为进入点。

#### 1.5.3. 整合构建命令

上面说的针对于linux系统的linker，然而对于其他系统不太适用。为解决这个问题，我们可以创建名为`.cargo/config.toml`的文件，并加入平台具体的参数如下：

```toml
# in .cargo/config.tom

[target.'cfg(target_os = "linux")']
rustflags = ["-C", "link-arg=-nostartfiles"]

[target.'cfg(target_os = "macos")']
rustflags = ["-C", "link-args=-e __start -static -nostartfiles"]
```

`rustflags`键包含了每次对`rustc`调用时自动加入的参数。

### 1.6. 小结

综上，一个最小的free standing的rust库如下

`src/main.rs`:

```rust
#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`Cargo.toml`
```rust
[package]
name = "crate_name"
version = "0.1.0"
authors = ["Author Name <author@example.com>"]

# the profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# the profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic
```

使用如下命令进行交叉编译

```shell
cargo build --target thumbv7em-none-eabihf
```

## 2. Freestanding/Baremetal Rust

### 2.1. about libcore, liballoc, libstd

+ libcore: 无依赖的rust核心库(core)，无libc，无heap
  + 要求`panic`和`eh_personality`
+ liballoc: 智能指针和堆管理的集合(即，Box)
  + 要求`global_allocator`和`alloc_error_handler`
  + 在有了基于物理内存页的受管理的堆内存后能够使用
+ libstd: rust软件的共有抽象的集合(例如，I/O，网络，线程)
  + 依赖于`libcore`和`liballoc`

### 2.2. `no_std`

无默认的prelude和标准库

### 2.3. `no_main`

+ 无`fn main()`定义（即，没有明确定义进入点）
+ linker默认的进入点是`_start()`

```rust
#![no_std]

#[no_mangle]
pub extern fn _start() -> ! {
    // ...
}
```

### 2.4. `panic_handler`

```rust
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
  // you will be asked to fill this in lab3
  loop {}
}
```

### 2.5. `eh_personality`

填补了系统和语言对于异常处理的语义差异

```rust
#[lang = "eh_personality"]
fn eh_personality() {
  // ignored for now!
}
```

### 2.6. 关于`.elf`和`.bin`

`.elf`是可执行目标文件，仍需要loader装载到内存，并可能存在load time的动态链接，因此文件中仍存在一些符号信息、重定位条目等。

`.bin`可以不需要loader提供的一些其他功能，只需放置到内存即可从entry point开始执行。不存在额外信息，因而大小也更小。自己写的bare-metal内核可以先生成elf文件，然后使用objcopy转为binary。

具体到rust，可以在生成.elf后使用

```rust
cargo objcopy -- --strip-all -O binary
```

## 3. reference

+ [A Freestanding Rust Binary](https://os.phil-opp.com/freestanding-rust-binary/)
+ [Freestanding/Baremetal Rust](https://tc.gts3.org/cs3210/2020/spring/l/lec07/lec07.html#cover)

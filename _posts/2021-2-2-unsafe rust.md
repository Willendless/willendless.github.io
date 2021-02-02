---
layout: "post"
author: LJR
title: unsafe rust
category: 编程语言
mathjax: true
tags:
    - rust
---

首先，如果主动使用`unsafe`关键字则代表编程者完全了解对应的风险并且需要自己保证内存安全性。

## 1. 心智模型

1. safe代码应当能够无条件依赖`unsafe`代码。
2. 相反`unsafe`应当足够鲁棒，尤其是涉及泛型以及和周围状态交互时。

## 2. 一些rust认为是安全但并不希望出现的情况

1. 泄露内存 -> 浪费内存，但没有悬垂指针
2. 整型溢出 -> 溢出但没有内存相关的违反
3. 获得原始指针 -> 只要不解引用都是安全的
4. 死锁 -> hanging，但没有crashing
5. 静态条件 -> 不一致状态，但是没有内存问题

### 2.1. 内存泄露

```rust
let s1 = String::from("Hello world!");
std::mem::forget(s1);
//=> no reclaiming of the memory as it own't call drop()

let s2 = Box::new(0xdeadbeef as u64);
Box::leak(s2);
//=> ditto
```

如上，`std::mem::forget`会释放`s1`内存，但是其对应的底层系统资源不会被关闭。一个应用场景是rust代码将某个文件描述符交给外部的c代码。

`Box::leak`会返回可变引用，但是owner同时变成unreachable。通常用于返回的引用需要在剩下整个程序的生命周期存活。如果返回的引用被drop，则会造成内存泄露。

### 2.2. 获取原始指针

```rust
let x = 1;
let y = &x as *const i32;
unsafe {
    assert_eq!(y.read_volatile(), 1);
}
unsafe {
    y.write_volatile(2);
}
assert_eq!(x, 2);
```

MMIO时，需要进行编译器认为unsafe的解引用操作

```rust
let y = 0xdeadbeef as *const i32;
unsafe {
    y.read_volatile();
    // => Program received signal SIGSEGV (fault address 0xdeadbeef)
}
```

访问原始指针时，编译器依赖于编程人员

+ 指针指向内存区域的liveness（i.e. 没有被freed）
+ 指针指向内存区域的ownership（i.e. 没有被reallocated）

### 2.3. 整型溢出

+ $+, -, *$能上溢/下溢
  + debug模式->panic
  + release模式->2的补码wrap
+ $<<, >>$能够移位超过自身类型位数
  + debug模式->panic
  + release模式->类型位数取模
+ $/, \%$ $INT_MIN-1$ panic

#### 2.3.1. 整型溢出的处理

类似于[GCC内置](https://gcc.gnu.org/onlinedocs/gcc/Integer-Overflow-Builtins.html)尽可能明确操作。

```rust
// a + b:
//  in debug  : a.checked_add(b).unwrap()
//  in release: a.wrapping_add(b)

fn overflowing_add(self, rhs: i64) -> (i64, bool);
fn wrapping_add(self, rhs: i64) -> i64;
fn saturating_abs(self) -> i64;

fn checked_add(self, rhs: i64) -> Option<i64>;
```

### 2.4. 内联汇编

```rust
#![feature(asm)]

fn add(x: u64, y: u64) -> u64 {
    let rtn;
    unsafe {
        asm!("add $0, $2"
             : "=r"(rtn)
             : "0"(x), "r"(y)
             :
             : "intel");
    }
    return rtn
}

dbg!(add(1, 2));
```

include一个汇编文件

```rust
// src/init.rs
global_asm!(include_str!("init/init.s"));

#[no_mangle]
unsafe fn kinit() -> ! {
    zeros_bss();
    kmain();
}
```

### 2.5. 全局可变静态变量

```rust
static mut GLOBAL: i64 = 0;

fn main() {
    unsafe {
        dbg!(GLOBAL);
    }
}
```

### 2.6. union访问

```rust
union Buf {
  int: u32,         // either u32
  buf: [u8; 4],     // or a byte array
}

fn main() {
  let u = Buf {
    int: 0xdeadbeef,
  };

  unsafe {
    dbg!(u.int);
    dbg!(u.buf);
  }
}
```

用union标识Atag

```rust
#[repr(C)]
pub struct Atag {
  pub dwords: u32,
  pub tag: u32,    // TAG indicates the type of 'Kind'
  pub kind: Kind
}

#[repr(C)]
pub union Kind {
  pub core: Core,
  pub mem: Mem,
  pub cmd: Cmd
}
```

### 2.7. FFI

```rust
extern crate libc;

use std::ffi::CString;
use libc::{size_t, c_char, c_void, c_int};

extern {
  fn malloc(size: size_t) -> *mut c_void;
  fn printf(format: *const c_char, ...) -> c_int;
}

fn main() {
  unsafe { malloc(100); }

  let fmt = CString::new("hello: %d\n").unwrap();
  unsafe {
    printf(fmt.as_ptr(), 1);
  }
}
```

### 2.8. Transmute

`transmute<T, U>`将一个为`T`类型的值重新以类型`U`解释它。注意，`T`和`U`的类型应当相同。

```rust
let s1 = "Hello world!";
let s2 = unsafe {
  std::mem::transmute::<&str, (u64, u64)>(s1)
};

dbg!(s1); //=> "Hello world!"
dbg!(s2); //=> (ptr-to-hello-world, 12)
```

`transmute`对应move语义，因此`drop()`不会被调用。

```rust
let s1 = String::from("Hello world!");
let s2 = unsafe {
  // s1 moves, and no drop() will be called in transmute()
  std::mem::transmute::<String, (u64, u64, u64)>(s1)
};
dbg!(s2);

unsafe {
  // not dangled
  printf(s2.0 as *const c_char);
}
```

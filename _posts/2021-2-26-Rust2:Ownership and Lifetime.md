---
layout: "post"
title: Rust（二）Ownership and Lifetime
author: LJR
category: 编程语言
tags:
    - rust
---

## 1. 所有权

所有权系统，即ORBM(*Ownership Based Resource Management*，基于所有权的资源管理)。要求资源和变量紧耦合，资源同一时间只能存在一个owner，owner负责自己管理的资源的释放。rust引入所有权的意义在于强制使用RAII管理资源。

rust的所有权机制是借助类型系统在**编译期实现**的。

### 1.1. move语义和所有权转移

move语义的解释：被move的资源保持不变，前一个owner的栈上内存被按位(by-bits)复制给后一个owner的栈上内存空间。同时，前一个owner的生命周期结束，处于uninitialized状态。

+ move语义通过`=`运算符显式转移所有权。
+ 或者通过传参隐式转移给函数参数。
+ 或者通过函数返回值隐式返回。

![rust1.png](https://i.loli.net/2021/02/26/mkjBCz8ey1q9pwL.png)

### 1.2. copy语义

copy语义的解释：前一个owner以及它拥有的资源保持不变。前一个owner的栈上存储空间被按位(by-bits)clone一份后给后者，拥有的资源也被clone一份给后者，同时修改后者栈上空间的指针。

![rust2.png](https://i.loli.net/2021/02/26/s2E4IdHFBPSJtkw.png)

默认情况下，变量的绑定具有move语义。除非实现`std::marker::Copy`trait，例如原始类型。

## 2. 生命周期

我的理解是：程序中有三个需要讨论生命周期的元素，即变量、引用和资源。强调rust生命周期这个概念的意义在于更好地理解rust要求程序遵守的约束，保证内存安全和线程安全。

rust的生命周期要求的规则也是借助编译器和borrow checker在**编译期**实现。

### 2.1. 变量

对于初始化了的变量，它的生命周期即它所在的*local scope*，当超出*scope*，需要释放:

1. 自己绑定的内存空间
2. 自己管理的资源。

这和所有权系统一致。

### 2.2. 引用

引用的生命周期需要满足下列两个约束：

+ **引用的生命周期不能超过其引用的值的所有者**。
+ **引用的生命周期应当大于等于存储其值的变量的生命周期**。

通过这两个约束，给出了引用生命周期的上下界。这样才能没有悬垂指针，内存安全。除此之外为了线程安全，引用还需要满足

+ **多个不可变引用 xor 一个可变引用**。

但这和生命周期关系不大。解决的是数据竞争的问题。

### 2.3. 资源

这里的资源指的是例如：堆内存，socket，文件描述符等可以move的实体，**不包括和初始化的变量绑定的内存空间（栈上内存）**。

### 2.4. 示例

不变式：*引用的生命周期应当大于或等于其引用的变量的生命周期，这样才不会出现悬垂指针。*

给出如下记号

+ 每一个变量和引用都有对应的生命周期
  + `x['a]`: x的生命周期是`'a`
  + `&'a`: 引用某个生命周期至少是`'a`的变量

#### 2.4.1. 示例一

```rust
'a: {
  let x['a]: i32 = 0;
  'b: {
    // y's lifetime is 'b
    let y['b]: &'a i32 = &x['a];
    //                   &x's lifetime is 'a (or shorter)

    // -> y can only contain a reference that lives longer than itself
    }
  }
}
```

#### 2.4.2. 示例二

+ borrow checker总是会采用尽可能小的生命周期

```rust
// 示例1
'a: {
  let x['a]: i32 = 0;
  'b: {
    let y['b]: &['a -> 'b] i32 = &x['a];
    // -> y can only contain a reference that lives longer than itself
    //    reduce its lifetime from 'a to 'b
    }
  }
}
// 示例2
let x = 0;
let y = &x;
let z = &y;
'a: {
  let x: i32 = 0;
  'b: {
    let y: &'b i32 = &['a -> 'b] x;
    'c: {
      let z: &'c &'b i32 = &['b -> 'c] y;
    }
  }
}
```

#### 2.4.3. 示例三

```rust
let x = 0;
let z;
let y = &x;
z = y;

// 尝试1
'a: {
  let x['a]: i32 = 0;
  'b: {
    let z['b]: &'b i32;
    'c: {
      let y['c]: &'c i32 = &['a -> 'c] x;
      
      // WARNING: &'b i32 <- &'c i32
      //  z contains a reference that lives shorter than itself!
      z = y;
    }
  }
}
// 尝试2
'a: {
  let x['a]: i32 = 0;
  'b: {
    let z['b]: &'b i32;
    'c: {
      // in fact, &x can be 'b or even 'a!
      let y['c]: &'b i32 = &'b x;

      // Okay. &'b i32 <- &'b i32
      z = y;
    }
  }
}
```

如上，`y`的生命周期为`'c`，而其包含的引用的生命周期是`'b`(最小化采用`b`而非`a`，不变式为:$c<=b<=b<a$)。

#### 2.4.4. 使用不变式判断

```rust
// 例1：
let y = {
  let x = String::from("Hello");
  &x
};
'a: {
  let y['a] = 'b: {
    let x['b] = String::from("Hello");
    &'b x
  };
  // WARNING. y['a] contains a reference that lives shorter (&'b)!
}
// 例2： 函数调用和返回
fn to_string(data: &u32) -> &String {
  let s = format!("{}", data);
  &s
}
fn to_string<'a>(data: &'a u32) -> &'a String {
  'b: {
    let s['b] = format!("{}", data);
    &'b s
    // WARNING: &'a <- &'b
    //  the return value contains a reference that lives shorter
  }
}
```

#### 2.4.5. 最小化生命周期的原因

以下是合法的几个例子。

```rust
fn main() {
  let mut s = String::from("Hello");
  let r1 = &mut s;
  let r2 = &s;

  dbg!(r2);
}
```

```rust
let mut s = String::from("Hello");
let c = s.as_str();
dbg!(c);
    
s.push_str(" World!");

// desugar之后
'a: {
  let mut s = String::from("Hello");
  'b: {
    let c = String::as_str(&'b s);
    dbg!(c);
   }
  'c: {
    String::push_str(&'c mut s, &'c " World!");
  }
}
```

可以发现，c的作用域从`let c`开始，一直到末尾，然而c的声明周期仅为`'b`。

### 2.5. 函数/方法的生命周期标注

```rust
fn bad_fn<'a>() {
    let _x = 12;
    let y: &'a i32 = &_x;
}
```

上述代码，`bad_fn`会自动引入作用域，且`bad_fn`生命周期不能超过`'a`

### 2.6. 结构体的生命周期标注

```rust
struct Borrowed<'a>(&'a i32);
```

表示，结构体中对`i32`的引用的生命周期应当超过结构体实例。


## 3. Owned type: &str vs. String

类型不一定具有明确的size（表示成`?Sized`），这意味着它们：

+ 无法copy/move到内存中
+ 它们的值仅能通过引用获取(`&T`)
+ 它们对应有owner类型(例如`String`是`str`的拥有者类型)，又被称为容器类型(*container type*)

以下是一些比较：

+ `&str`和`String`：`String`是容器类型能够执行push，扩充内存等操作，`&str`仅是切片类型，指向的资源长度一定。
+ `&str`和`&String`：`&String`是指向容器类型的引用，`String`作为容器类型，位于栈上空间。`&str`是对堆上空间的切片引用。
+ `str`和`[u8; N]`：`str`长度不定，`[u8; N]`是数组，长度一定，可以存在于栈上。
+ `str`和`[u8]`：本质上一样
+ `&str`和`&[u8]`：本质上一样

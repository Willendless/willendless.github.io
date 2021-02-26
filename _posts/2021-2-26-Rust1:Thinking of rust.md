---
layout: "post"
title: Rust（一）Thinking of rust
author: LJR
category: 编程语言
tags:
    - rust
---

## 1. RAII: Resource Acquisition Is Initialization/*Scope-Bound* Resource Management

使用**局部对象**来管理资源。

RAII的意义在于将那些使用前需要先获取的资源(例如：堆内存，socket，文件，锁，数据库连接等)的生命周期和对象的生命周期绑定。

RAII保证了在任意能够访问到该对象的函数中能够访问对应的资源（资源的可获得性是“类不变式，class invariant，约束类的对象”，消除了冗余的运行时判断）

RAII可以被总结为

+ 将资源封装在类中
  + 类的构造器负责获取资源和建立所有类不变式，当无法获取资源时抛出异常
  + 析构器释放资源且不会抛出异常
+ 总是通过RAII-类的实例访问资源
  + 资源本身为automatic storage或者生命周期是temporary 
  + 资源的生命周期被本地变量的生命周期限定

### 1.1. scoping rules and ownership

所有权规则指的是

+ RAII类实例在其生命周期内拥有资源（实例位于**local, function scope**）
+ 当类实例被回收时，同时回收其拥有的资源

因而也被称为*Ownership Based Resource Management(OBRM)*。

### 1.2. scoping rules and lifetime

"scoping"并非一定是*local scope*，因而实际上被称为*lifetime*。

```c++
std::ofstream *file = new std::ofstream("/etc/passed");
...
// @different
//    function
//    thread
//    long long time later
//    or even when the process dies
//    ...
delete file;
```

## 2. Substructure Type System

+ 内存~=资源：内存也被视为资源，需要基于所有权规则管理内存
+ 实现方式：通过增强类型系统在编译期静态约束`malloc()`/`free()`接口的使用
  + 例如：变量初始化：给予内存的所有权
  + 例如：超出scope，调用`free()`

内存资源的含义：**和变量绑定的内存以及变量各个域绑定的内存**。

## 3. Immutability and Mutability

+ 变量默认为不可变

```rust
//
//      +--> variable name (or identifier)
//      |
let mut x: i32 = 1;
//  +--    ---   +-> value
//  |      |
//  |      +--> type (32-bit signed integer)
//  +--> noting mutability
//
x = 2;
```

+ 变量和类型：变量是一块内存的拥有者
+ 值和类型：值也具有类型
+ 强类型语言：赋值时变量的类型和值的类型必须匹配

## 4. Ownership and Borrowing

+ 变量**owns**一个值：拥有者负责内存的回收
+ move语义：通过赋值运算符`=`，变量的值move给另一个变量，如下

```rust
{
    // x <- alloc("Hello")
    let mut x = String::from("Hello!");
    // y <- alloc("World!")
    let y = String::from("World!");
    // drop(x): free("Hello!")
    x = y;
    ...
}
// drop(x); free("World!")
```

![]()

## 5. Safety rule: {shared} + xor mutable references

+ 变量从owner借用值
+ Rule 1：能够存在任意共享引用(*shared references*)
+ Rule 2：仅能存在一个可变引用(*mutable reference*)
+ Safety rule: Rule 1 **xor** Rule 2

### 5.1. 意义1：别名分析

更容易分析指针别名 => 容易优化

```rust
// $ man 3 memcpy
//
// NOTES
//   Failure to observe the requirement that the memory 
//   areas do not overlap has  been  the  source of
//   significant bugs.
//
void *memcpy(void *dst, const void *src, size_t n);

// dst and src can not point to the same value!
fn memcpy<T>(dst: &mut T, src: &T);
```

### 5.2. 意义2：无数据竞争

+ 无法同时存在读者和写者
+ 实际上，该规则过于严格，其实一些不符合规则的惯例也是安全的

## 6. 小结：Rust中的伟大思想

**OBRM plus {shared} + ^ mutable references**

+ 内存安全
+ 线程安全

加上现代的类型系统，抽象，构建系统，包管理器等等。

### 6.1. 关于变量和值的一点想法

我的理解是，变量和值是有必要区分的。变量从所有权规则的角度考虑较好，值从生命周期的角度考虑较好。

变量需要满足**所有权规则**，即负责其对应的资源的获取和回收。当然资源可以move。

资源具有生命周期，资源的生命周期应当大于等于其的拥有者。因为可以move。

那这么说的话其实**所有权规则**和**生命周期**是从两个角度共同对程序做出的约束。

## 7. reference

+ [RAII, cppreference.com](https://en.cppreference.com/w/cpp/language/raii)

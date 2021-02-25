---
layout: "post"
author: LJR
title: 读《Rusty Types for Solid Safety》笔记
category: 编程语言
tags:
    - rust
---

> what you don't use, you don't pay for. what you do use, you couldn't hand code any better.

## 类型安全

如果一个程序不存在未定义行为，则认为该程序是良定义的(well-defined)。如果一个语言的安全性检查能保证每一个用其写的程序是良定义的，则认为该语言是类型安全的。安全性检查可能发生在运行时也可能发生在编译期。

## 内存安全

不存在对未定义的内存的访问。

- 空间安全（spatial safety）：没有越界(out of bounds)访问。
- 时间安全（temporal safety）：在对象生命周期前后不存在对该对象的访问。

## rusty类型

### linearity

即所有权系统(ownership)。

owner和move语义：对每一个给定对象仅能存在一个绑定，对象的所有权通过`=`赋值符号传递。

### Memory Safety and Race Freedom

对每一个存储位置需要保证满足以下其中一点：

+ 1个可变引用和0个不可变引用
+ 0个可变引用和任意个不可变引用

以上不变式保证了不会同时存在读者和写者。但是还不够满足内存安全，试想对某个已经move的变量的引用将成为悬垂指针。因此还需要

+ 对象的引用需要通过owner传递性地创建(即变量从所有者处借用值)，类型系统需要确保在引用还有效时对象的owner不能改变。

以上保证了不存在悬垂指针或指向无效内存的指针。

```rust
let mut x = Vector([1, 2, 3]);
let y = &x[0];
clear(&mut x);
let z = *y;
```

### (Re)borrowing

borrow即创建引用。reborrow即通过已经存在的引用创建引用。

```rust
let mut x = V(...);
let a = &mut x; // x: 不可用；*a：可写，a：可读
let b = &(*x); // *a: 可读，{*b，b}可读
```

我的理解是reborrow类似于从租客手中租房子（转租）。

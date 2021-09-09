---
layout: "post"
title: Rust（六）Smart Pointer and Container
author: LJR
category: 编程语言
tags:
    - rust
---

> Smart pointers behave as if it owns (or points to) the underlying data.

为模仿指针的功能智能指针至少应当重载下面两个操作：

1. 解引用: `*ptr`，即实现`Deref/DerefMut`
2. 析构: 当`ptr`超出作用域，即实现`Drop`

## 1. Rc: 引用计数指针

+ `Rc<T>`使得可以共享数据的所有权

### 1.1. 什么时候使用`Rc`?

1. 在递归数据结构中使用，以避免冗余存储
2. 难以在编译期分析生命周期

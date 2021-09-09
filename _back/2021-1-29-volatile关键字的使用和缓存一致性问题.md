---
category: "post"
author: LJR
title: volatile关键字的使用
category: 编程语言
tags:
    - c/c++
---

最近面试中碰到了`volatile`的使用问题，在这里总结一下。

`volatile`主要会在下面三种情况中使用

1. 内存映射的外设寄存器
2. 被中断例程修改的全局变量
3. 多线程程序中的关键区变量

## 1. 外设寄存器

外设寄存器的值相对于程序流来说是异步改变的。



## 2. 中断服务例程

TODO

## 3. 多线程引用

TODO: 真的能这么用吗

## 4. java中的volatile关键字

## 5. 参考

+ [Jones, Nigel. "Introduction to the Volatile Keyword" Embedded Systems Programming, July 2001](https://barrgroup.com/embedded-systems/how-to/c-volatile-keyword)
+ [volatile与内存屏障总结](https://zhuanlan.zhihu.com/p/43526907)

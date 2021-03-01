---
layout: "post"
title: Rust（五）Generics and Traits
author: LJR
category: 编程语言
tags:
    - rust
---

和传统的OOP语言通过类封装数据和行为不同，Rust的类型和行为是相互区分的，不同的数据类型通过trait相互关联。

## 1. Generics Overview

泛型版本的`sum()`如下，即参数化多态/模板：

```rust
fn sum(args: &[T]) -> T
where
    T: Default + AddAssign + Copy,
{
    let mut v: T = Default::default();
    for &a in args {
        v += a;
    }
    return v;
}
```

### 1.1. 泛型的实现

通过两种实现物化泛型

```rust
dbg!(sum(&[1, 2, 3, 4]));
dbg!(sum(&[1.0, 2.0, 3.0, 4.0]));

# test2::sum()
0000000000013e00 <_ZN5test23sum17h76c0ce64971bc830E>:
...
# test2::sum()
0000000000013ea0 <_ZN5test23sum17h8f28e3dbc449aa4aE>:
...
```

### 1.2. 示例：实现`Point`

```rust
#[derive(Default, Copy, Clone, Debug)]
struct Point(i64, i64);

impl AddAssign for Point {
    fn add_assign(&mut self, rhs: Self) {
        self.0 += rhs.0;
        self.1 += rhs.1;
    }
}
dbg!(sum(&[Point[0, 0], Point(10, 10)]));
```

泛型版本的`Point`

```rust
#[derive(Default, Copy, Clone, Debug)]
struct Point<T>(T, T);

// 特殊化：为实现了AddAssign trait的参数类型实现AddAssign trait
impl<T: AddAssign> AddAssign for Point<T> {
    fn add_assign(&mut self, rhs: Self) {
        self.0 += rhs.0;
        self.1 += rhs.1;
    }
}
dbg!(sum(&[Point[0.0, 0.0]), Point(10.0, 10.0)]);

#[derive(Default, Copy, Clone, Debug)]
struct Point<T1, T2>(T1, T2);

impl<T1: AddAssign, T2: AddAssign> AddAssign for Point<T1, T2> {
    fn add_assign(&mut self, rhs: Self) {
        self.0 += rhs.0;
        self.1 += rhs.1;
    }
}

dbg!(sum(&[Point(0, 0.0), Point(10, 10.0)]));
```

## 2. Trait Overview

+ 区分数据和行为
+ Rust同时提供函数的静态/动态分发
  + 能够选择是否使用虚函数表(Trait object)

### 2.1. 示例：Summary trait

```rust
trait Summary {
    fn summary(&self) -> String;
}

impl<T1: Debug, T2: Debug> Summary for Point<T1, T2> {
    fn summary(&self) -> String {
        format!("Point({:?}, {:?})", self.0, self.1)
    }
}

impl<T> Summary for Vec<T> {
    fn summary(&self) -> String {
        format!("Vec({}/{})", self.len(), self.capacity())
    }
}
```

### 2.2. 静态/动态分发

```rust
// 静态分发
fn summarize0<T: Summary>(v: &T) {
    dbg!(v.summary());
}
// 或者写成
fn summarize1(v: &impl Summary) {
    dbg!(v.summary());
}
// 通过虚函数表动态分发
fn summarize2(v: &dyn Summary) {
    dbg!(v.summary());
}
```

上述对应的汇编如下

```shell
lea    rax,[rip+0x234b4]        # Point(1, 2)
mov    rdi,rax
call   4340 <test2::summarize0>

lea    rax,[rip+0x234a5]        # Point(1, 2)
mov    rdi,rax
call   45f0 <test2::summarize1>

lea    rax,[rip+0x23496]        # Point(1, 2)
lea    rcx,[rip+0x2f423]        # a virtual function table
mov    rdi,rax
mov    rsi,rcx
call   48a0 <test2::summarize2>
```

虚函数表：

![v-table.png](https://i.loli.net/2021/03/01/a2fTl5Kmk6RSOUo.png)


## 3. 常用的trait

### 3.1. formatting相关

+ fmt::Debug -> `{:?}`
+ fmt::Display -> `{}`
+ fmt::Write -> formatting的低级接口

### 3.2. IO reader/writer

+ io::Read
+ io::Write

无buffer。但是能够在其上增加其他功能，例如`BufReader`

```rust
fn main() -> std::io::Result<()> {
    let f = File::open("log.txt")?;
    let mut reader = BufReader::new(f);

    let mut line = String::new();
    let len = reader.read_line(&mut line)?;
    println!("First line is {} bytes long", len);
    Ok(())
}
```

### 3.3. Traits和Rust语言是紧耦合的

+ `Add`,`AddAssign`,etc: `+`,`+=`,etc
+ `Iterator`和`IntoIterator`: for循环
+ `Deref`和`DerefMut`: `.`(dot)

### 3.4. `From`->`Into`

```rust
pub trait Into<T> {
    fn into(self) -> T;
}
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

### 3.5. Iterator和IntoIterator

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item=Self::Item>;
    fn into_iter(self) -> Self::IntoIter;
}
```

下面给出为`Point`实现迭代器的例子:

```rust
struct IterPoint<T>(Point<T, T>, T);

impl Iterator for IterPoint<i32> {
    type Item = Point<i32, i32>;
    fn next(&mut self) -> Option<Self::Item> {
        let rtn = Some(self.0);
        (self.0).0 += self.1;
        (self.0).1 += self.1;
        rtn
    }
}

impl IntoIterator for Point<i32, i32> {
    type Item = Point<i32, i32>;
    type IntoIter = IterPoint<i32>;
    fn into_iter(self) -> Self::IntoIter {
        IterPoint(self, 1)
    }
}

let line = Point(0,0).into_iter();
let even = line.step_by(2);
let quad = even.filter(|x| x.0 % 4 == 0)

for i in quad.take(10) {
    dbg!(i);
}
```

注：`Self`(S大写)表示实现了当前trait的具体类型。方法的第一个参数的`self`类型默认是`self: Self`。

## 4. 参考

+ [Generics and Traits](https://tc.gts3.org/cs3210/2020/spring/l/lec08/lec08.html)

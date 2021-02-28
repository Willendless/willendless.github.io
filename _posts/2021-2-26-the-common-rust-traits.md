---
layout: "post"
title: The Common Rust Traits
author: LJR
category: 编程语言
tags:
    - rust
---

这篇文章主要内容来自[The Common Rust Traits](https://stevedonovan.github.io/rustifications/2018/09/08/common-rust-traits.html)。

## 1. What is a Trait?

在Rust中，数据类型 - 原始类型，结构体，枚举类型和其他聚合类型例如元组和数据本身都非常单一。尽管也可以为它们实现方法，但是方法仅是函数的变体。类型之间并不存在关系。

而引入*Traits*这种抽象机制的目的就在于为类型增加功能，同时构建类型之间的关系。具体来说有下面两种运作模式

+ 接口：*traits*支持接口继承，但不是实现继承。
+ 泛型约束：*traits*被用于泛型约束。泛型函数定义于实现了具体traits的类型之上，也即避免了c++模板的“compile-time duck typing”。如果我们传参传的是一只鸭子，那么它必须实现`Duck`。仅仅有`quack()`方法是不够的。

## 2. Converting Things to Strings

考虑定义了`to_string`方法的`ToString`trait。为了传入实现该trait的类型的引用作为参数，有两种写法。

第一种是通过泛型或者说单态(*monomorphic*):

```rust
use std::string::ToString;

fn to_string1<T: ToString> (item: &T) -> String {
    item.to_string()
}
println!("{}", to_string1(&42));
println!("{}", to_string1(&"hello"));
```

item是到某个实现了`ToString`的类型的引用。

第二种是通过动态或者说多态(*polymorphic*):

```rust
fn to_string2(item: &ToString) -> String {
    item.to_string()
}
println!("{}", to_string2(&42));
println!("{}", to_string2(&"hello"));
```

第一种情况，类似于c++的模板，编译器会为不同类型生成不同的代码。这种方法最为高效，`to_string`可以是内联的。

第二种情况，代码只被生成一次，但是实际的`to_string`被动态调用。这里的`&ToString`类似于java的接口或者c++带虚方法的基类。

对某个类型的引用被称为*trait object*。*trait object*含有两部分：

+ 原始的引用
+ 虚方法表：包含了trait的方法

```rust
let d: &Display = &10;
```

对于*trait object*，rust采用一个更加明确的表示`&dyn ToString`。

## 3. Printing Out: Display and Debug

为了能够通过`{}`格式化打印某个值，需要实现`Display`trait。而`{:?}`要求实现`Debug`trait。

```rust
use std::fmt;

#[derive(Debug)]
struct MyType {
    x: u32,
    y: u32
}

impl fmt::Display for MyType {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "x={},y={}", self.x, self.y)
    }
}

let t = MyType{x:1,y:2};
println!("{}", t);
```

`write!`宏和`println!`关系紧密，其第一个参数是任何实现了`Write`trait的类型。

大多数标准库类型都实现了`Debug`，且能够给出开发者友好的字符串表示。

任何实现了`Display`trait的类型会自动实现`ToString`，因此`42.to_string()`，`"hello".to_string()`也能工作。

## 4. Default

`Default`trait给出某个类型的默认值，例如：0是数值的默认值，空向量是vectors的默认值，""是`String`的默认值。

一种间接生命整型变量并设置为0的方法如下，`default`是一个返回`T`的泛型方法。因此rust需要知道`T`的类型：

```rust
let n: u64 = Default::default();
```

`Default`也是通过derive实现。

```rust
#[derive(Default)]
struct MyStruct {
    name: String,
    age: u16
}
let mine: MyStruct = Default::default();
```

**Rust喜欢明确，因而对变量初始化并赋值为默认值不会自动发生。**例如，`let n: u64;`Rust会预期在之后初始化，而不会自动初始化。

rust中不存在"named function parameters"（即python中类似于`f(a=1,b=2)`。但是有一种惯用法能够达到相同的目的，例如有一个函数需要非常多配置参数，因此你定义了一个结构体称为`Config`，若`Config`实现了`Default`，则调用函数时就不用说明`Config`的每一个域，而是使用下面的方法

```rust
my_function(Config {
    job_name: "test",
    output_dir: Path::new("/tmp"),
    ...Default::default()
})
```

## 5. Conversion: From and Into

`From`给出使用`from`方法将某个值转换为另一个的方法。因而可以使用`String::from("hello")`。如果`From`被实现了，那么`Into`trait会被自动实现。

因为`String`实现了`From<&str>`，`&str`自动实现了`Into<String>`

```rust
let s = String::from("hello");
let s: String = "hello".into();
```

一个例子如下，json对象使用字符串作为索引，新的域可以被创建然后插入`JsonValue`值中：

```rust
obj["surname"] = JsonValue::from("Smith");
obj["name"] = "Joe".into();
obj["age"] = 35.into();
```

这里因为rust能够推断出左值的类型，使用`into`方法更加方便（更容易写和阅读）。

`From`表示了一种总是能够成功的类型转换。然而它开销较大：将字符串切片转换为`String`需要分配buffer并按字节拷贝。

`From/Into`在Rust错误处理时经常会使用，对于`Result<T,E>`:

```rust
let res = returns_some_result()?
```

上面`?`运算符实际上是下面代码的语法糖：

```rust
let res = match return_some_result() {
    Ok(r) => r,
    Err(e) => return Err(e.into())
}
```

即一个error类型能够转换为能被返回的error类型`E`。

一种有用的错误处理策略是让函数返回`Result<T, Box<Error>>`。任何实现了`Error`trait的类型都能够被转换为trait object`Box<Error>`。

## 6. Making Copies: Clone and Copy

`From/Into`描述了不同得类型是如何互相转换得。`Clone`描述了相同类型得一个新值是如何被创建的。Rust喜欢让所有可能开销很大的操作比较明显，因此需要使用`val.clone()`。

让类型能够clone很方便，只需要

```rust
#[derive(Debug, Clone)]
struct Person {
    first_name: String,
    last_name: String,
}
```

`Copy`是一个*marker trait*(不存在需要实现的方法)，

```rust
#[derive(Debug,Clone,Copy)]
struct Point {
    x: f32,
    y: f32,
    z: f32
}
```

当然只有所有域都实现了`Copy`，才能为该类型实现`Copy`。

## 7. Fallible Conversions - FromStr

某些情况下的类型转换会存在错误，例如：整数42转换为字符串可以使用`ToString`trait定义的`to_string`方法。然而若是从"42"转换为`i32`类型则就是fallible转换。

这种转换的方法由`FromStr`提供，需要实现者：

1. 定义`from_str`方法
2. 设置当转换失败时需要返回的关联类型`Err`

通常，该trait隐含地通过`parse`方法使用，该方法具有泛型的返回值，因而需要使用turbofish运算符表明返回值的类型：

```rust
let answer = match "42".parse::<i32>() {
    Ok(n) => n,
    Err(e) => panic!("'42' was not 42!");
}
```

或者使用`?`，更加优雅一些:

```rust
let answer: i32 = "42".parse()?;
```

Rust标准库为数值类型和网络地址定义了`FromStr`。

## 8. Reference Conversions - AsRef

`AsRef`用于两种类型的引用之间相互转换的情况，且转换的开销相对较少。

最常用的情景是和`&Path`一起使用。Rust使用专门的`PathBuf`存储文件系统路径名，底层使用的是`OsString`(用于存储不被信任的操作系统字符串)。`&Path`是`PathBuf`的引用。而从常规的Rust字符串获取`&Path`引用开销很小。

```rust
// asref.rs
fn exists(p: impl AsRef<Path>) -> bool {
    p.as_ref().exists()
}

assert!(exists("asref.rs"));
assert!(exists(Path::new("asref.rs")));
let ps = String::from("asref.rs");
assert!(exists(&ps));
assert!(exists(PathBuf::from("asref.rs")));
```

使用`impl AsRef<Path>`处理文件系统路径的函数或方法能够以任何实现了`AsRef<Path>`的类型作为参数。根据文档

```rust
impl AsRef<Path> for Path
impl AsRef<Path> for OsStr
impl AsRef<Path> for OsString
impl AsRef<Path> for str
impl AsRef<Path> for String
impl AsRef<Path> for PathBuf
```

`String`也实现了`AsRef<str>`，因而可以使用

```rust
fn is_hello(s: impl AsRef<str>) {
    assert_eq("hello", s.as_ref());
}
is_hello("hello");
is_hello(String::from("hello"));
```

但是，rust程序员一般使用`&str`作为字符串参数的类型，并能通过`deref coercion`机制传参。

## 9. Overloading `*` - Deref

Rust的很多字符串方法并非直接定义在`String`上。`String`的方法通常会修改字符串，例如`push`和`push_str`。但是类似于`starts_with`的函数也能被应用在字符串切片上。

`Deref`trait用于实现"dereference"操作符`*`。**这和c语言中的解引用具有相同的语义：从引用指向的内存中提取出值。**若`r`是一个引用，则有`r.foo()`，但是如果你想获取值，就必须使用`*r`。

`Deref`最常见的用例是被使用在智能指针例如`Box<T>`和`Rc<T>`中。智能指针表现的像对其内存储值的引用，因而它们可以对`Box<T>`调用`T`的方法。

`String`实现了`Deref`。若`s`类型是`String`则`&*s`的类型是`&str`。

*Deref coercion*意味着`&String`会被隐含地转换为`&str`:

```rust
let s: String = "hello".into();
let rs: &str = &s;
```

然而，`&String`是一个和`&str`不同的类型。当使用`match`运算符明确地匹配类型时，仍必须使用`s.as_str()`，`&s`在这里无法工作

```rust
let s = "hello".to_string();
match s.as_str() {
    "hello" => {....},
    "dolly" => {....},
}
```

Deref coericion也被用于决议方法，若某个方法不是定义在`String`上，则可以尝试使用`&str`。这表现地就像有限形式的继承。

除了`String`和`&str`，`Vec<T>`和`&[T]`也具有类似关系。

## 10. Ownership: Borrow

`String`"own"它们的数据，像`&str`这样的类型能够从owned类型"borrow"数据。

`Borrow`解决了一个maps和sets的问题。通常我们会把owned字符串保存在`HashSet`中以避免borrowing规则。

```rust
let mut set = HashSet::new();
set.insert("one".to_string());
// set is noew HashSet<String>
if set.contains("two") {
    println!("got two!");
}
```

`Borrow`trait使得能使用`&str`类型的值查询set或maps。和`AsRef`不同

+ `Borrow`的实现要求更严格，会要求owned和borrowed的值的hash和ordering值相同
+ `Borrow`为上面的集合数据结构提供了blanket实现`T:Borrow<T>`.`AsRef`提供了另一种不同实现，基本上只要`T: AsRef<U>`就有`&T: AsRef<U>`。

## 11. I/O: Read and Write

`std::fs::File`和`std::io::Stdin`不同。Rust并没有将stdin视作文件的一种。它们相同之处在于实现了trait`Read`。

基本的`read`方法读取一些字节到buffer中，并返回`Result<usize>`。

`Read`提供了`read_to_string`方法，其会读取整个文件作为`String`。`read_to_end`方法读取整个文件作为`Vec<u8>`(若不能保证文件是utf-8编码的，使用`read_to_end`更好)。

`Read`不是Rust prelude的一部分。可以使用`use std::io::prelude::*;`获取所有I/O traits。

**Rust I/O 默认是unbuffered的。**

例如，若想以最快的可能方式从stdin中读取内容，首先lock它

```rust
let stdin = io::stdin();
let mut lockin = stdin.lock();
// lockin is buffered
```

Locked stdin实现了`ReadBuf`（定义了buffered reading）。`lines()`方法迭代遍历输入的所有行，但是它为每一行分配一个新的字符串，因而效率很低。**为了最佳的性能，使用`read_line`，因为它允许重用单个字符串buffer。**

类似地，为了从文件中获取buffered reading：

```rust
let mut rdr = io::BufReader::new(File::open(file)?);
```

注意，Rust默认不适用buffered io的目的在于让buffering和allocation更加明确。

对于`Write`trait，文件、sockets和标准流（stdout和stderr）实现了它。同样，它是unbufferd且`io::BufWriter`用于为实现了`Write`的类型增加buffering。

**为了避免不同线程产生相互干扰的输出，`println`宏会获取唯一的锁，这导致较低的性能。若需要高性能，使用buffer和`write`宏。**

## 12. Iteration: Iterator and IntoInterator

`Iterator`方法只需要实现一个`next`方法，其返回值类型是`Option`。

使用迭代器的一种罗嗦的方法是

```rust
let mut iter = [10, 20, 30].iter();
while let Some(n) = iter.next() {
    println!("got {}", n);
}
```

`for`语句的使用更加精简

```rust
for n in [10, 20, 30].iter() {
    println!("got {}", n);
}
```

`for`语句需要提供的表达式其实是：“任何能够转换为迭代器的东西”，即由`IntoIterator`描述。所以，`for n in &[10, 20, 30] {...}`也能正常工作 - slice实现了`IntoIterator`。上面的代码因为迭代器本身也实现了`IntoIterator`。

因此，Rust的`for`语句和一个trait紧密相关。

Rust中迭代器属于0开销抽象。事实上，如果明确地通过索引循环访问切片会更慢，因为rust会添加运行时的索引检查。

`Iterator`的提供方法有很多默认实现，例如`map`，`filter`。使用它们可以避免写出循环，例如求和：

```rust
let res: i64 = (0..n).into_iter().sum();
```

**传递一系列值给函数的通用方式是使用`IntoIterator`**。使用`&[T]`限制太多而且要求调用者构建buffer(不方便且开销大):

```rust
fn sum(ii: impl IntoIterator<Item=i32>) -> i32 {
    ii.into_iter().sum()
}
println!("{}", sum(0..9));
println!("{}", sum(vec![1, 2, 3]));
```

## 13. Conclusion: Why are there So Many Ways to Create a String?

```rust
let s = "hello".to_string(); // ToString
let s = String::from("hello"); // From
let s: String = "hello".into(); // Into
let s = "hello".to_owned(); // ToOwned
```

有非常多方法能够创建一个字符串。但是值得注意的是**这些方法都不是`String`类型本身的方法**。它们所对应的`trait`都是必须的，因为它们让泛型编程得以工作。**When you create strings in codde, just pick one way and use it consistently.**

Rust的traits在使用前需要先引入作用域中。例如，在某个实现了`Error`trait的类型调用`description()`之前需要使用`use std::error::Error`。

---
title: Rust的字符串
date: 2019-11-21 00:00:00
categories:
- Rust
---

### Rust中str、&str、String和&String的区别和关系

<!-- more  -->

先看看Rust创建字符串String常用的三种方法：

```rust
let s1 = "hello".to_string();
let s2 = String::from("hello");
let s3 = format!("hello");
```

凭我们的直观"hello"本来就是一个字符串了，为什么还需要这么复杂的？也许这样更好：

```rust
let s1 = "hello";
```

但在clion中会提示我们s是一个&str类型，Rust出于什么原因做这样的区分？这两个类型又有什么不同？

* str是存在内存中的动态长度的utf-8不可变序列，由于大小未知<?>，因此只能使用指针处理它，使用&str访问值

* &str对某些utf-8数据的引用，通常称为"字符串切片"，它只是某些数据的视图，数据可以存储在任何地方：

  * 静态存储：字符串字面量"hello"是&'static str，程序运行时，数据被硬编码到可执行文件并加载到内存中

    ```rust
    let s1 = "hello";
    ```

  * 指向堆上的String：对String类型变量进行切片，&str获取其数据视图（借用）

    ```rust
    let s2 = &String::from("hello")[..]
    ```

  * 栈

    ```rust
    use std::str;

    let s3: &str = str::from_utf8(&[b'h', b'e', b'l', b'l', b'o']).unwrap();
    ```

* String是具有所有权的动态字符串，分配在堆上，和Vec类似，当你需要拥有或修改数据时使用它

* &String是对于String的借用，本质上是一个指针。通常可以被强制隐式转换为&str，所以很少直接使用

  > 使用&String的一种情况：将可变引用传递给需要修改字符串的函数
  >
  > ```rust
  > fn main() {
  >     let mut s = String::from("Hello, Rust!");
  >     foo(&mut s);
  > }
  >
  > fn foo(s: &mut String) {
  >     s.push_str("appending foo..");
  >     println!("{}", s);
  > }
  > ```

参考资料：

* <https://www.ameyalokare.com/rust/2017/10/12/rust-str-vs-String.html>
* <http://llever.com/gentle-intro/pain-points.zh.html>
* <https://blog.thoughtram.io/ownership-in-rust/>

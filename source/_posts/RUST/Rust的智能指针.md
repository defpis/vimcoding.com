---
title: Rust的智能指针
date: 2019-11-21 00:00:00
categories:
- Rust
---

### 引用、指针和智能指针

* Rust的引用就是指针，通过&符号借用它们指向的值。除了引用数据外，它们没有任何其他特殊功能，也没有任何开销。
* 智能指针是充当指针的特殊数据结构，它具有相较于普通指针额外的元数据和拓展功能。
* 在Rust中，标准库定义的不同智能指针所提供的功能超出了引用所提供的功能，我们可以通过它们完成许多特定的功能。

<!-- more -->

### Rust标准库中的智能指针`Box<T>`、`Rc<T>`和`RefCell<T>`介绍

* `Box<T>`在栈上存储指针，在堆上存储数据。用来存储编译时无法确定大小的数据类型，例如引用自身的枚举定义：

  ```rust
  enum List {
      Cons(i32, List),
      Nil,
  }
  ```

  Rust尝试计算List大小时，会陷入无限递归！通过`Box<T>`包裹可以安全的通过编译：

  ```rust
  enum List {
      Cons(i32, Box<List>),
      Nil,
  }
  ```

* `Rc<T>`跟踪数据的引用数量。用来共享数据，实现多重所有权。

  ```rust
  let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
  let b = Cons(3, Box::new(a));
  let c = Cons(4, Box::new(a));
  ```

  上面的代码很显然会抛出借用错误，因为转移了两次`a`的所有权，但是如果需求上非要满足`b`和`c`的后继节点是`a`，且它们都需要有`a`的所有权，那么不通过一点特殊的手段，在当前借阅规则下就无法实现。

  ```rust
  enum List {
      Cons(i32, Rc<List>),
      Nil,
  }
  
  use crate::List::{Cons, Nil};
  use std::rc::Rc;
  
  fn main() {
      let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil))))); // count = 1
      let b = Cons(3, Rc::clone(&a)); // count++
      let c = Cons(4, Rc::clone(&a)); // count++
  }
  // c count--
  // b count--
  // a count--
  // count = 0
  ```

  通过`Rc<T>`对数据的不可变引用计数，当引用计数为`0`时，离开作用域会触发drop析构。

> `Rc<T>`仅适用于不可变引用，如果你想要一个线程安全的替代品，请使用`std::sync::Arc`

* `RefCell<T>`实现内部可变性，并在运行时检查借阅规则。

  通常情况，给定一个不可变值，无法获取它的可变引用，下面的代码无法通过编译：

  ```rust
  let x = 5;
  let y = &mut x;
  ```

  通过`RefCell<T>`就可以在运行时获取不可变值的可变引用

  ```rust
  #[derive(Debug)]
  enum List {
      Cons(Rc<RefCell<i32>>, Rc<List>),
      Nil,
  }
  
  use crate::List::{Cons, Nil};
  use std::rc::Rc;
  use std::cell::RefCell;
  
  fn main() {
      let value = Rc::new(RefCell::new(5));
  
      let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));
  
      let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
      let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));
  
      *value.borrow_mut() += 10; // 5 + 10 = 15
  
      println!("a after = {:?}", a);
      // a after = Cons(RefCell { value: 15 }, Nil)
      println!("b after = {:?}", b);
      // b after = Cons(RefCell { value: 6 }, Cons(RefCell { value: 15 }, Nil))
      println!("c after = {:?}", c);
      // c after = Cons(RefCell { value: 10 }, Cons(RefCell { value: 15 }, Nil))
  }
  ```

> Rust借阅规则？
>
> * At any given time, you can have *either* (but not both of) one mutable reference or any number of immutable references.
> * References must always be valid.
>
> 编译时/运行时检查借阅规则的区别
>
> With references and `Box`, the borrowing rules’ invariants are enforced at compile time. With `RefCell`, these invariants are enforced *at runtime*. With references, if you break these rules, you’ll get a compiler error. With `RefCell`, if you break these rules, your program will panic and exit.

### 使用`Deref`特质将智能指针视为常规引用

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0 // MyBox通过元组构造，取第一个元素的引用
    }
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y); // *(y.deref())只会发生一次，不会无限递归
}
```

### 使用`Drop Trait`运行清理代码

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };
    let d = CustomSmartPointer { data: String::from("other stuff") };
    println!("CustomSmartPointers created.");
}
```

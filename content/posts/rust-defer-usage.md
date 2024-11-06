---
title: rust 的 defer 几个使用场景
date: 2024-11-06
tag:
  - rust
---

```rust
pub trait Deref {
    type Target: ?Sized;

    // Required method
    fn deref(&self) -> &Self::Target;
}
```

`Defer` 这个 `trait` 有一个关联类型，`defer` 这个函数返回的是关联类型的引用，而不是取消引用的值，这一点可能会造成误解。

可以这样来理解：给定类型 `T`, 为 `T` 实现 `Defer` 是让编译器可以把 `&T` 引用为 `&Target`，对使用者面言，操作包装器跟操作实际被包装的类型一样，简化操作。

## 使用场景：隐式调用 `Target` 的字段或者方法

```rust
struct Point { x: f32, y: f32 }

fn main() {
    let boxed: Box<Point> = Box::new(Point { x: 5, y: 10 });
    println!("X: {}", boxed.x);
}
```

`boxed` 并没有 `x` 字段，但是能正常编译运行。 这是因为编译器首先会先查看 `boxed` 是否有 `x` 字段， 如果没有，则会检测是否实现的 `Defer`，再看 `<T as Defer>::Target` 有没有 `x` 的字段。

所以 `boxed.x` 实际是 `boxed.defer().x`，编译器可以让我们省略这个 `.defer()` 的调用，简化操作，像直接操作 `Point` 一样。

## 使用场景：借用操作

```rust
fn main() {
    let boxed: Box<i32> = Box::new(5);
    print(&boxed);
}

fn print(input: &i32) {
    println!("Input: {input}");
}
```

这里编译器会检查 `&boxed` 是否是类型 `&i32`，如果不是，也会检查是否实现了 `Defer`，然后检查 `&Target` 是否是 `&i32`。

所以 `print(&boxed)` 实际是 `print(boxed.defer())`，这个 `&` 借用操作隐式调用了 `defer()`。

## 使用场景：* 指针操作符

`*` 操作是与 `&` 相互的， `*` 是取值。

```rust
fn main() {
    let boxed: Box<&str> = Box::new("hello world");
    let deref_result: &str = *boxed;
}
```

需要注意，这个 `*` 改变的是 `Target` 的值，如：
```rust
fn main() {
    let mut boxed: Box<Vec<i32>> = Box::new(vec![2, 5]);
    *boxed = vec![3, 4];

    // 编译过不去，类型不匹配
    *boxed = Box::new(vec![3, 5]);
}
```

## 使用场景： 隐式链式调用

```rust
fn main() {
    let boxed = Box::new(Box::new(Box::new("hello world")));
    let v = &boxed;
    println!("{v}");
}
```
任意类型只要实现了 `Defer`，都可以被这样链式隐式调用 `defer()`，编译器做了这些检测，方便我们操作，而不是显示这样 `T.defer().defer()...`。

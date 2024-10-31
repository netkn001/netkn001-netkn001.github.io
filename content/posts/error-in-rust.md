---
title: rust 中的错误处理
date: 2024-10-27
tags:
  - rust
---

一个项目一般由多个组件组成，每个组件都有自己的 `Error` 定义，那怎么定义呢？没有标准的错误规范。

一个好的错误处理其实是不容易的，需要考虑多个方面，既要对用户友好，还要对开发友好。

良好的错误处理需要考虑：
1. 如何定义错误
2. 如何记录错误
3. 如何呈现错误

## rust 中的 Error

`rust` 的错误处理以 `Result<T, E>` 枚举来表示，其中 `E` 通常（但并非必要）扩展 `std::error::Error`。

```rust
enum Result<T, E> {
   Ok(T),
   Err(E),
}

pub trait Error: Debug + Display {
    // Provided methods
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
    fn description(&self) -> &str { ... }
    fn cause(&self) -> Option<&dyn Error> { ... }
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... }
}
```

看 `std::error::Error` 这个 `trait` 并不复杂，因此想实现一种自定义错误类型还是挺容易的。但是， 自定义类型一般没这么做，随着我们自定义的错误类型越来越多，这些样板代码会一直重复，也难以使用泛型来示，宏也不容易。

## rust 中的错误处理生态

为了简化自定义的错误处理，`thiserror` 和 `anyhow` 就是做这件事，这两个库，`thiserror` 更适用于库，`anyhow` 则更适用于二进制文件，这些库就可以简便错误处理，直接使用 `?` 来传播错误。

但是，在实际使用中，`thiserror` 和 `anyhow` 也是有些问题：
1. `anyhow context` 虽然可以包含上下文来区分具体错误，但是这是返回统一的 `anyhow::Error`，另外通过上下文来区分具体错误也不是一个好主意，如果要针对特定的错误做不同的处理就不好办了。
2. `thiserror` 是实现 `std::convert::From` 这个 `trait`，所以不能出现两个相同源类型，会导致上下文模糊，如在 `i/o` 操作，读写文件都是 `io` 错误，这样就不知道是具体那边代码的问题了。

`snafu` 这个 `crate` 像是 `thiserror` 和 `anyhow` 的结合体，`thiserror` 提供了一个方便的宏来定义自定义错误类型，包括显示、源和一些上下文字段。 `anyhow` 提供了一个 `Context` 特征，可以轻松地将一个潜在错误转换为具有新上下文的另一个错误。 关键是 `snafu` 可以出现相同的源错误。 当然，`snafu` 有自己的约定替换，遵守即可。

## 好的错误处理

### 1. 定义错误

自定义的处理通常用 `enum` 来表示，利用语言特性，很方便统一处理，而不至于有些错误没有被处理到。然后可以对应错误码来统一。

```rust
#[derive(Debug, Snafu)]
enum Error {
    #[snafu(display("error A"))]
    ErrorA,
    #[snafu(display("error B{id}"))]
    ErrorB { id: u64 },
    #[snafu(display("error C"))]
    ErrorC {
        source: io::Error,
        location: Location,
    },
    #[snafu(whatever, display("{message}"))]
    ErrorOther {
        message: String,
        #[snafu(source(from(Box<dyn std::error::Error>, Some)))]
        source: Option<Box<dyn std::error::Error>>,
    },
}

impl Error {
    fn code(&self) -> i32 {
        match self {
            Error::ErrorA => 10000,
            Error::ErrorB { .. } => 100001,
            Error::ErrorC { .. } => 20000,
            Error::ErrorOther { .. } => 90000,
        }
    }
}
```

### 记录错误

这里主要是面向开发者，快速定位问题。

一般都是直接记录 `error` 的错误信息，如 `DecodeMessage(serde_json: invalid character at 1)` 或者加了一些上下文如 `unmarshal xxxx - DecodeMessage(serde_json: invalid character at 1)`，这种如果只在一个地方出现还好，但是一般会在多次地方出现，这就导致了不知道直接有问题的代码在哪了。因此，需要增加更多的信息来记录。

首先能想到的就是增加出现错误的行列号了(`file!(), line!(), colomn!()`)这三个宏，`snafu` 的 `location!()`，是的，加上这个就可以准确定位了。

如果想做得更好就要像系统的堆栈那样了，有完整的上下文和调用链，但是这个是有损耗的，`release` 环境是不能直接用的。另外完整的堆栈也包含了很多无用的信息（系统堆栈等），需要自己跳过不相关的代码来查到应用程序的调用栈。

那有没有办法做到轻量级的堆栈，只保留应用程序中的堆栈呢？

// TODO: 待补充

### 呈现错误

开发人员，直接看日志即可。但是用户层上呢，暴露错误码是一个做法（如上示例），类似 `windows` 的错误码体系。还可以自己做用户提示的映射。

利用 `rust` 的特性，不会漏处理任何的错误分支。

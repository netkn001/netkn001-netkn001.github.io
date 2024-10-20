---
title: 使用 unwarp 的三种形式
date: 2024-10-20
tags:
  - rust
---

>我称之为我的十亿美元错误……
>
>当时，我正在设计第一个全面的类型系统，用于面向对象语言中的引用功能。我的目标是确保所有对引用的使用都是绝对安全的，由编译器自动执行检查。
>
> 但是我无法拒绝定义一个 `Null` 引用的诱惑，因为它实在太容易实现了。这导致了无数错误、漏洞和系统崩溃。在过去的四十年里，这些问题可能已经造成了十亿美元的损失。
>
> —— Tony Hoare，ALGOL W 的发明者。

`rust` 是消除空指针的语言，或者带有 `Result` 和 `Option` 类似的类型语言，或者显示返回错误如 `go`，都是为了消除空指针的问题。

但是，我们在编写代码中，真的能规避空指针的问题吗？以 `rust` 为例，很多 `unwrap` 的操作是导致另一种形式的空指针。

这里的三种 `unwrap` 形式不是指具体的方法调用如 `unwrap_or()`, `unwrap_or_default()`, `unwrap_or_else()` 等等，是指按场景来分类。

1. 作为显示 `panic!` 操作
2. 作为一定不会到达的操作，类似 `unreachable!()`
3. 作为后面在优化的临时操作(个人主观的备忘)

其中第三点应该在整个程序生命周期中都存在，也是大家最最最常写的方式。

# unwrap as panic!()

这在初始化代码最为常见，也是应该的做法，因为初始化的操作也是程序启动后第一件要做的事情。如果失败，程序也不能正常使用。

这是正确的做法。

一个 `web server` 的例子：

```rust
let app = Router::new().route("/", get(get_info));
let address_str = format!("{address}:{port}");

let addr: SocketAddr = address_str.parse().unwrap();

let listener = TcpListener::bind(&addr).await.unwrap();

axum::serve(listener, app.into_make_service()).await.unwrap();
```

# unwrap as unreachable!()

这种方式比较不明显，是明确一定不会执行到的代码，在静态初始化的时候比较常见。

如下示例：
```rust
static HEXADECIMAL_REGEX: LazyLock<Regex> = LazyLock::new(|| Regex::new("^[0-9a-f]*$").unwrap());
```

# unwrap as TODO

这里是用 `// TODO` 注释来表示，而不是用 `todo!()` 这个宏， 因为 `todo!()` 是等效于直接`panic!()`。

**这种形式很多受我们主观的想法来操作**，我们在写代码过程中， 总是急于验证代码的“是否正常即是否可运行”的做法。 所以很多时候是直接使用 `unwrap()` 来直接解包，而忽略边界条件，然后构造的数据又总是是正常的数据。（这里忽略单元测试的做法）。

好一点的可能会在对应代码块上备注注释 `// TODO 临时处理，后续优化 blabla`。至少可能后面还会优化，但是并不总是会真的优化，如果在测试环境中运行良好。 又如果 `unwrap()` 没有备注，或者多人协作没有规范，就成了隐藏的炸弹，代码的健壮性受到威胁。

针对这种主动行为，我们可以：
1. 善用 `Result`, `Option` 提供的方法。
2. 看情况使用 `unrwap_or`, `unwrap_or_else`, `unrwap_or_default` 等官方做法。
3. 将 `unwrap` 转换为错误处理，并增加上下文，如 `anyhow` 提供的 `context`。

多点样板处理代码，增加代码的健壮性。

---
title: 关于内存序，做个小记
date: 2014-10-19
tags: ["rust", "system"]
---

在之前写使用 `go` 的时候，使用 `atomic` 很简单，如示例：

```go
var flag int32
atomic.StoreInt32(&flag, 1)
atomic.LoadInt32(&flag)
//...
```
隐藏了内存序的指定。

在使用 `rust` 的时候，这个内存序是必须要指定的

```rust
static X: AtomicI32 = AtomicI32::new(0);
X.fetch_add(5, Relaxed);
X.load(Relaxed);
```
对于没有详细了解这个内存序，很是懵逼。

内存序(Memory Ordering) 是代码编译成机器指令后的执行顺序，在多线程程序中，是对共享内存的读写执行顺序的约定。 确保了在并发环境下，线程之间对共享数据访问不会导致不一致性和不可预测的行为。内存序主要涉及以下几个方面：
1. 可见性：一个线程对共享数据的修改，何时能被其他线程看到。
2. 顺序性：不同线程之间对共享数据访问的顺序是否保持一致。
3. 原子性：确保某些操作在执行时不会被其他线程干扰。

内存序问题产生的主要原因：
- 编译器优化： 编译器为了提高程序性能，不会完全按照代码顺序生成机器指令，会进行指令重排做优化。
- 处理器优化：`CPU` 在运行时为了提高性能，还可能对指令重新排序。

在 `rust` 的原子操作(`std::sync::atomic`)中，定义了如下类型：

```rust
pub enum Ordering {
    Relaxed,
    Release,
    Acquire,
    AcqRel,
    SeqCst,
}
```
- **Relaxed**: 只保证原子性，不保证顺序，简单理解就是最终一致性，如全局递增计数就很适合。
- **Release**: 确保当前操作(如 `store`)之前的所有写入都完成, 对后续的读(如 `load`)操作可见，与 `Acquire` 配合使用。
- **Acquire**: 确保当前操作(如 `load`)之后的所有读取都能看到之前的写入，与 `Release` 配置使用。
- **AcqRel**:  结合了 `Release` 和 `Acquire` 的特性。
- **SeqCst**: (Sequential Consistency)提供最强的一致性保证，确保全局顺序，会增加程序的同步开销，强制编码器和硬件保持操作的顺序，需要更多的同步机制来保证。

虽然有 5 个枚举值，但实际上是 3 类内存排序，由弱到强
1. Relaxed
2. Release, Acquire, AcqRel
3. SeqCst

这三类不能混合使用。

那 `go` 为什么没有暴露这个内存序的参数出来呢？官方的文档 [atomic](https://pkg.go.dev/sync/atomic) 有说明要非常小心才能正确使用，所以默认是使用 `SeqCst` 的方式，然后不暴露这个参数，保持简单。

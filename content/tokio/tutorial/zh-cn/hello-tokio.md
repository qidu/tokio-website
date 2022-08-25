---
title: "Hello Tokio"
---

我们将从写一个非常基本的Tokio应用开始。用它连接Mini-Redis服务端，设入<'hello','world'>。
再读出键值。这里会使用Mini-Redis客户端库。

# 代码

## 创建新crate

先生成Rust应用:

```bash
$ cargo new my-redis
$ cd my-redis
```

## 添加依赖

接着打开 `Cargo.toml` 添加如下内容到 `[dependencies]`:

```toml
tokio = { version = "1", features = ["full"] }
mini-redis = "0.4"
```

## 写代码

接着打开 `main.rs` 并替换这个文件的内容成如下:

```rust
use mini_redis::{client, Result};

# fn dox() {
#[tokio::main]
async fn main() -> Result<()> {
    // Open a connection to the mini-redis address.
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Set the key "hello" with value "world"
    client.set("hello", "world".into()).await?;

    // Get key "hello"
    let result = client.get("hello").await?;

    println!("got value from the server; result={:?}", result);

    Ok(())
}
# }
```

确认先启动 Mini-Redis 服务端. 在另一个终端窗口执行:

```bash
$ mini-redis-server
```

如果还没有安装 mini-redis, 可用如下命令: 

```bash
$ cargo install mini-redis
```

现在运行 `my-redis` 应用:

```bash
$ cargo run
got value from the server; result=Some(b"world")
```

Success!

完整代码在 [这里][full].

[full]: https://github.com/tokio-rs/website/blob/master/tutorial-code/hello-tokio/src/main.rs

# 分解过程

先这里做了什么，虽然没有太多代码，但也发生了好多隐含过程执行。

```rust
# use mini_redis::client;
# async fn dox() -> mini_redis::Result<()> {
let mut client = client::connect("127.0.0.1:6379").await?;
# Ok(())
# }
```

函数 [`client::connect`] 由 `mini-redis` crate提供。它异步的建立到目标地址的TCP连接。
一旦连接成功，返回一个 `client` 句柄。尽管这个操作执行是异步的，但从代码上看起来却像是同步的。
`.await` 是这些异步操作的唯一提示。

[`client::connect`]: https://docs.rs/mini-redis/0.4/mini_redis/client/fn.connect.html

## 什么是异步编程?

大多数电脑程序执行顺序与代码编写顺序相同。执行第一行，然后下一行，直到结束。在同步程序中，当遇到一个操作
无法立即完成，它将阻塞等待。例如，建立一个TCP连接时需要与网络对端交换信令，这将耗费一些时间，在这段时间里，执行线程将被阻塞住。

异步编程时, 那些无法立即完成的操作都被挂起暂停在后台，前台线程不会被阻塞，能够继续往下执行。一旦阻塞的操作完成后，任务被唤起，
继续从原有位置执行。我们的例子中仅有一个任务，所以挂起时没有别的需要执行。异步程序中通常有许多任务并发执行。 

虽然异步编程能够让程序响应更快，它也导致了更复杂的程序结构。程序员需要跟踪所有任务状态以恢复那些完成了的异步操作。 曾经这是
最容易出错的工作。

## 编译期绿色线程

Rust通过[`async/await`]特性实现异步编程.异步操作函数被标记上`async`关键字。例子中的 `connect` 函数即是:

```rust
use mini_redis::Result;
use mini_redis::client::Client;
use tokio::net::ToSocketAddrs;

pub async fn connect<T: ToSocketAddrs>(addr: T) -> Result<Client> {
    // ...
# unimplemented!()
}
```

这里 `async fn` 定义看起来虽然像同步函数, 但执行起来是异步的。Rust在编译时将`async fn` 标记的函数转换到一个异步过程中执行。
对`async fn`函数调用 `.await` 让出前台线程的控制权。前台线程继续执行其他过程，阻塞过程转入后台线程。

[[警告]]
| 虽然其他语言也实现了 [`async/await`] , Rust使用了更独特的方法。
| Rust的async操作是**lazy**的. 这与其他语言运行时的语义不同。

[`async/await`]: https://en.wikipedia.org/wiki/Async/await

如果还不理解也别担心，我将在剩下的章节继续探讨 `async/await`特性。

## 使用 `async/await`

异步函数被调用时很像其他Rust函数。但调用异步函数并不是立即执行函数体，相反，调用`async fn`函数时返回一个代表函数体的值. This is conceptually
在概念上类似于没有参数的闭包（closure）。要执行函数体，则需要在返回值上进一步调用`.await`操作符。

例如，

```rust
async fn say_world() {
    println!("world");
}

#[tokio::main]
async fn main() {
    // Calling `say_world()` does not execute the body of `say_world()`.
    let op = say_world();

    // This println! comes first
    println!("hello");

    // Calling `.await` on `op` starts executing `say_world`.
    op.await;
}
```

输出:

```text
hello
world
```

调用`async fn`函数的返回值是一个匿名类型，它实现了[`Future`] trait.

[`Future`]: https://doc.rust-lang.org/std/future/trait.Future.html

## 异步 `main` 函数

The main function used to launch the application differs from the usual one
found in most of Rust's crates.

1. It is an `async fn`
2. It is annotated with `#[tokio::main]`

An `async fn` is used as we want to enter an asynchronous context. However,
asynchronous functions must be executed by a [runtime]. The runtime contains the
asynchronous task scheduler, provides evented I/O, timers, etc. The runtime does
not automatically start, so the main function needs to start it.

The `#[tokio::main]` function is a macro. It transforms the `async fn main()`
into a synchronous `fn main()` that initializes a runtime instance and executes
the async main function.

For example, the following:

```rust
#[tokio::main]
async fn main() {
    println!("hello");
}
```

gets transformed into:

```rust
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello");
    })
}
```

The details of the Tokio runtime will be covered later.

[runtime]: https://docs.rs/tokio/1/tokio/runtime/index.html

## Cargo features

When depending on Tokio for this tutorial, the `full` feature flag is enabled:

```toml
tokio = { version = "1", features = ["full"] }
```

Tokio has a lot of functionality (TCP, UDP, Unix sockets, timers, sync
utilities, multiple scheduler types, etc). Not all applications need all
functionality. When attempting to optimize compile time or the end application
footprint, the application can decide to opt into **only** the features it uses.

For now, use the "full" feature when depending on Tokio.

---
title: 术语
---

## Asynchronous 异步

在Rust上下文中, 异步代码是指那些使用了async/await语言特性的代码块, 这使很多任务可以并发的运行在少量线程（甚至单个线程中）。

## Concurrency and parallelism 并发与并行

并发（Concurrency）与并行（parallelism）都是同时执行多任务相关的概念。如果一些事情并行发生，也可以说它们是并发的。但相反，
如果在两个任务中交替执行，但从没有实际上在同一时刻执行，就只能称之为并发，而非并行。

## Future 将来

Future是保存了一些操作块的当前状态的值。它有 `poll` 函数，能够让这些操作继续执行，直到需要等待什么状态，
比如一个网络连接的结果。调用`poll` 方法，应该非常快就返回。

Futures可以通过在异步代码块（async block）中使用`.await` 组合很多其他futures。

## Executor/scheduler 执行器/调度器

执行器/调度器就是能够重复调用futures的`poll` 方法的模块。在标准库中并没有执行器，需要额外的库。
最广泛被使用的执行器就是由Tokio 运行时提供的。

执行器能够在少量线程上并发运行大量的futures，它依靠将运行中的任务遇到awaits时及时交换出去。
如果一段代码执行很长时间没有遇到`.await`，说明它阻塞了线程或没让出资源给执行器，这将阻碍其他任务的执行。

## Runtime 运行时

A runtime is a library that contains an executor along with various utilities
运行时是一个集成了各种功能的执行器，比如计时组件、IO组件。名词运行时和执行器通常指能互换使用。
标准库并没有运行时，所以你需要额外的库来提供，Tokio就是被最广泛用到的一个。

提到运行时时，有时候也在其他上下文环境有别的意思，如 "Rust has no runtime" 有时是指Rust没有垃圾回收和JIT编译执行模式。

## Task 任务

任务是指在Tokio运行时上执行的一段代码。使用[`tokio::spawn`] 或 [`Runtime::block_on`] 函数来创建。
通过调用`.await` and [`join!`] 来合并创建futures的工具并不创建新任务，只是合并他们到已有的任务中。

被要求并行执行的多任务，也可能通过`join!`被并发的合并在单任务中执行。

[`tokio::spawn`]: https://docs.rs/tokio/1/tokio/fn.spawn.html
[`Runtime::block_on`]: https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html#method.block_on
[`join!`]: https://docs.rs/tokio/1/tokio/macro.join.html

## Spawning

Spawning is when the `tokio::spawn` function is used to create a new task. It
can also refer to creating new thread with [`std::thread::spawn`].

[`tokio::spawn`]: https://docs.rs/tokio/1/tokio/fn.spawn.html
[`std::thread::spawn`]: https://doc.rust-lang.org/stable/std/thread/fn.spawn.html

## Async block

An async block is an easy way to create a future that runs some code. For
example:

```
let world = async {
    println!(" world!");
};
let my_future = async {
    print!("Hello ");
    world.await;
};
```

The code above creates a future called `my_future`, which if executed prints
`Hello world!`. It does this by first printing hello, and then running the
`world` future. Note that the code above does not print anything on its own —
you have to actually execute `my_future` before anything happens, by either
spawning it directly, or by `.await`ing it in something you spawn.

## Async function

Similarly to an async block, an async function is an easy way to create a
function whose body becomes a future. All async functions can be rewritten into
ordinary functions that return a future:

```rust
async fn do_stuff(i: i32) -> String {
    // do stuff
    format!("The integer is {}.", i)
}
```

```rust
use std::future::Future;

// the async function above is the same as this:
fn do_stuff(i: i32) -> impl Future<Output = String> {
    async move {
        // do stuff
        format!("The integer is {}.", i)
    }
}
```

This uses [the `impl Trait` syntax][book10-02] to return a future, since
[`Future`] is a trait. Note that since the future created by an async block does
not do anything until it is executed, calling an async function does not do
anything until the future it returns is executed [(ignoring it triggers a
warning)][unused-warning].

[book10-02]: https://doc.rust-lang.org/book/ch10-02-traits.html#returning-types-that-implement-traits
[`Future`]: https://doc.rust-lang.org/stable/std/future/trait.Future.html
[unused-warning]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4faf44e08b4a3bb1269a7985460f1923

## Yielding

In the context of asynchronous Rust, yielding is what allows the executor to
execute many futures on a single thread. Every time a future yields, the
executor is able to swap that future with some other future, and by repeatedly
swapping the current task, the executor can concurrently execute a large number
of tasks. A future can only yield at an `.await`, so futures that spend a long
time between `.await`s can prevent other tasks from running.

To be specific, a future yields whenever it returns from the [`poll`] method.

[`poll`]: https://doc.rust-lang.org/stable/std/future/trait.Future.html#method.poll

## Blocking

The word "blocking" is used in two different ways: The first meaning of
"blocking" is simply to wait for something to finish, and the other meaning of
blocking is when a future spend a long time without yielding. To be unambiguous,
you can use the phrase "blocking the thread" for the second meaning.

Tokio's documentation will always use the second meaning of "blocking".

To run blocking code within Tokio, please see the [CPU-bound tasks and blocking
code][api-blocking] section from the Tokio API reference.

[api-blocking]: https://docs.rs/tokio/1/tokio/#cpu-bound-tasks-and-blocking-code

## Stream

A [`Stream`] is an asynchronous version of an [`Iterator`], and provides a
stream of values. It is commonly used together with a `while let` loop like this:

```
use tokio_stream::StreamExt; // for next()

# async fn dox() {
# let mut stream = tokio_stream::empty::<()>();
while let Some(item) = stream.next().await {
    // do something
}
# }
```

The word stream is confusingly sometimes used to refer to the [`AsyncRead`] and
[`AsyncWrite`] traits.

Tokio's stream utilities are currently provided by the [`tokio-stream`] crate.
Once the `Stream` trait is stabilized in std, the stream utilities will be moved
into the `tokio` crate.

[`Stream`]: https://docs.rs/tokio-stream/0.1/tokio/trait.Stream.html
[`tokio-stream`]: https://docs.rs/tokio-stream
[`Iterator`]: https://doc.rust-lang.org/stable/std/iter/trait.Iterator.html
[`AsyncRead`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html
[`AsyncWrite`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html

## Channel

A channel is a tool that allows one part of the code to send messages to other
parts. Tokio provides a [number of channels][channels], each serving a different
purpose.

- [mpsc]: multi-producer, single-consumer channel. Many values can be sent.
- [oneshot]: single-producer, single consumer channel. A single value can be sent.
- [broadcast]: multi-producer, multi-consumer. Many values can be send. Each
  receiver sees every value.
- [watch]: single-producer, multi-consumer. Many values can be sent, but no
  history is kept. Receivers only see the most recent value.

If you need a multi-producer multi-consumer channel where only one consumer sees
each message, you can use the [`async-channel`] crate.

There are also channels for use outside of asynchronous Rust, such as
[`std::sync::mpsc`] and [`crossbeam::channel`]. These channels wait for messages
by blocking the thread, which is not allowed in asynchronous code.

[channels]: https://docs.rs/tokio/1/tokio/sync/index.html
[mpsc]: https://docs.rs/tokio/1/tokio/sync/mpsc/index.html
[oneshot]: https://docs.rs/tokio/1/tokio/sync/oneshot/index.html
[broadcast]: https://docs.rs/tokio/1/tokio/sync/broadcast/index.html
[watch]: https://docs.rs/tokio/1/tokio/sync/watch/index.html
[`async-channel`]: https://docs.rs/async-channel/
[`std::sync::mpsc`]: https://doc.rust-lang.org/stable/std/sync/mpsc/index.html
[`crossbeam::channel`]: https://docs.rs/crossbeam/latest/crossbeam/channel/index.html

## Backpressure

Backpressure is a pattern for designing applications that respond well to high
load. For example, the `mpsc` channel comes in both a bounded and unbounded
form. By using the bounded channel, the receiver can put "backpressure" on the
sender if the receiver can't keep up with the number of messages, which avoids
memory usage growing without bound as more and more messages are sent on the
channel.

## Actor

A design pattern for designing applications. An actor refers to an independently
spawned task that manages some resource on behalf of other parts of the
application, using channels to communicate with those other parts of the
application.

See [the channels chapter] for an example of an actor.

[the channels chapter]: /tokio/tutorial/channels

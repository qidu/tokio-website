---
title: "Streams"
---

一个流是一系列的异步值。它等同于Rust的[`std::iter::Iterator`][iter]迭代每个异步操作，
用 [`Stream`] 特性来描述。流可以在 `async` 函数中迭代。它们也能转换成使用装饰器。
Tokio 在 [`StreamExt`] 特性里提供了许多装饰器。

<!--
TODO: bring back once true again?
Tokio provides stream support under the `stream` feature flag. When depending on
Tokio, include either `stream` or `full` to get access to this functionality.
-->

Tokio 在独立的 crate `tokio-stream` 中支持流。

```toml
tokio-stream = "0.1"
```

> 提示： 
> 目前 Tokio 的流工具类放在 `tokio-stream` crate 中。一旦 `Stream` 特性在Rust标准库中
> 稳定后，Tokio 的流工具类也将被移动到 `tokio` crate 中。

<!--
TODO: uncomment this once it is true again.
A number of types we've already seen also implement [`Stream`]. For example, the
receive half of a [`mpsc::Receiver`][rx] implements [`Stream`]. The
[`AsyncBufReadExt::lines()`] method takes a buffered I/O reader and returns a
[`Stream`] where each value represents a line of data.
-->

# 迭代器

目前，Rust语言还没支持异步的 `for` 循环。由此我们用`while let`循环和[`StreamExt::next()`][next]
一起来来迭代流。

```rust
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let mut stream = tokio_stream::iter(&[1, 2, 3]);

    while let Some(v) = stream.next().await {
        println!("GOT = {:?}", v);
    }
}
```

像迭代器一样， `next()` 方法返回 `Option<T>` 且 `T` 是流类型值。收到 `None` 暗示流迭代中止。

## Mini-Redis 广播

让我们看看一个稍微复杂的例子，用Mini-Redis客户端。

在 [这里][full] 有完整的代码。

[full]: https://github.com/tokio-rs/website/blob/master/tutorial-code/streams/src/main.rs

```rust
use tokio_stream::StreamExt;
use mini_redis::client;

async fn publish() -> mini_redis::Result<()> {
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Publish some data
    client.publish("numbers", "1".into()).await?;
    client.publish("numbers", "two".into()).await?;
    client.publish("numbers", "3".into()).await?;
    client.publish("numbers", "four".into()).await?;
    client.publish("numbers", "five".into()).await?;
    client.publish("numbers", "6".into()).await?;
    Ok(())
}

async fn subscribe() -> mini_redis::Result<()> {
    let client = client::connect("127.0.0.1:6379").await?;
    let subscriber = client.subscribe(vec!["numbers".to_string()]).await?;
    let messages = subscriber.into_stream();

    tokio::pin!(messages);

    while let Some(msg) = messages.next().await {
        println!("got = {:?}", msg);
    }

    Ok(())
}

# fn dox() {
#[tokio::main]
async fn main() -> mini_redis::Result<()> {
    tokio::spawn(async {
        publish().await
    });

    subscribe().await?;

    println!("DONE");

    Ok(())
}
# }
```

一个任务被生成以在 "numbers"键上发送消息到 Mini-Redis 服务端。在mian任务中，我们订阅
"numbers" 键并展示收到的消息。

订阅后，subscriber的[`into_stream()`] 被调用。这使用了 `Subscriber`，并返回一个能产生
到达消息的流。在我们开始迭代消息前，注意这个流被用[`tokio::pin!`]来[pin住][pin]在栈里。
在流上调用 `next()`需要它是被[pin住][pin]的。`into_stream()` 函数返回一个**不是**pin住
的流，为迭代它我们必须显式的pin它。

> 提示： 
> Rust值是"pinned"后不能在内存中移动。它的关键属性是指针能指向pinned值，调用者能相信指针
> 是有效的。这个特征被`async/await` 使用以支持跨`.await`点借用数据。

如果我们忘记pin流，将得到如下错误:

```text
error[E0277]: `from_generator::GenFuture<[static generator@Subscriber::into_stream::{closure#0} for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6> {ResumeTy, &'r mut Subscriber, Subscriber, impl Future, (), std::result::Result<Option<Message>, Box<(dyn std::error::Error + Send + Sync + 't0)>>, Box<(dyn std::error::Error + Send + Sync + 't1)>, &'t2 mut async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't3)>>>, async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't4)>>>, std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't5)>>, impl Future, Option<Message>, Message}]>` cannot be unpinned
  --> streams/src/main.rs:29:36
   |
29 |     while let Some(msg) = messages.next().await {
   |                                    ^^^^ within `tokio_stream::filter::_::__Origin<'_, impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`, the trait `Unpin` is not implemented for `from_generator::GenFuture<[static generator@Subscriber::into_stream::{closure#0} for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6> {ResumeTy, &'r mut Subscriber, Subscriber, impl Future, (), std::result::Result<Option<Message>, Box<(dyn std::error::Error + Send + Sync + 't0)>>, Box<(dyn std::error::Error + Send + Sync + 't1)>, &'t2 mut async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't3)>>>, async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't4)>>>, std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't5)>>, impl Future, Option<Message>, Message}]>`
   |
   = note: required because it appears within the type `impl Future`
   = note: required because it appears within the type `async_stream::async_stream::AsyncStream<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 'static)>>, impl Future>`
   = note: required because it appears within the type `impl Stream`
   = note: required because it appears within the type `tokio_stream::filter::_::__Origin<'_, impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`
   = note: required because it appears within the type `tokio_stream::map::_::__Origin<'_, tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>`
   = note: required because it appears within the type `tokio_stream::take::_::__Origin<'_, tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::take::Take<tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>>`
```

如果你遇到一个类似错误，尝试pin值。在运行前，先启动 Mini-Redis 服务器:

```text
$ mini-redis-server
```

然后运行例子，我们将在STDOUT上看到如下消息. 

```text
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"two" })
got = Ok(Message { channel: "numbers", content: b"3" })
got = Ok(Message { channel: "numbers", content: b"four" })
got = Ok(Message { channel: "numbers", content: b"five" })
got = Ok(Message { channel: "numbers", content: b"6" })
```

有些早的消息可能被丢弃，因为在订阅和发布之间有竞争。这个程序不会退出。一个到 Mini-Redis 管道
的订阅者将与服务端一样活跃。

Let's see how we can work with streams to expand on this program.

# Adapters

Functions that take a [`Stream`] and return another [`Stream`] are often called
'stream adapters', as they're a form of the 'adapter pattern'. Common stream
adapters include [`map`], [`take`], and [`filter`].

Lets update the Mini-Redis so that it will exit. After receiving three messages,
stop iterating messages. This is done using [`take`]. This adapter limits the
stream to yield at **most** `n` messages.

```rust
# use mini_redis::client;
# use tokio_stream::StreamExt;
# async fn subscribe() -> mini_redis::Result<()> {
#    let client = client::connect("127.0.0.1:6379").await?;
#    let subscriber = client.subscribe(vec!["numbers".to_string()]).await?;
let messages = subscriber
    .into_stream()
    .take(3);
#     Ok(())
# }
```

Running the program again, we get:

```text
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"two" })
got = Ok(Message { channel: "numbers", content: b"3" })
```

This time the program ends.

Now, let's limit the stream to single digit numbers. We will check this by
checking for the message length. We use the [`filter`] adapter to drop any
message that does not match the predicate.

```rust
# use mini_redis::client;
# use tokio_stream::StreamExt;
# async fn subscribe() -> mini_redis::Result<()> {
#    let client = client::connect("127.0.0.1:6379").await?;
#    let subscriber = client.subscribe(vec!["numbers".to_string()]).await?;
let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
        Ok(msg) if msg.content.len() == 1 => true,
        _ => false,
    })
    .take(3);
#     Ok(())
# }
```

Running the program again, we get:

```text
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"3" })
got = Ok(Message { channel: "numbers", content: b"6" })
```

Note that the order in which adapters are applied matters. Calling `filter`
first then `take` is different than calling `take` then `filter`.

Finally, we will tidy up the output by stripping the `Ok(Message { ... })` part
of the output. This is done with [`map`]. Because this is applied **after**
`filter`, we know the message is `Ok`, so we can use `unwrap()`.

```rust
# use mini_redis::client;
# use tokio_stream::StreamExt;
# async fn subscribe() -> mini_redis::Result<()> {
#    let client = client::connect("127.0.0.1:6379").await?;
#    let subscriber = client.subscribe(vec!["numbers".to_string()]).await?;
let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
        Ok(msg) if msg.content.len() == 1 => true,
        _ => false,
    })
    .map(|msg| msg.unwrap().content)
    .take(3);
#     Ok(())
# }
```

Now, the output is:

```text
got = b"1"
got = b"3"
got = b"6"
```

Another option would be to combine the [`filter`] and [`map`] steps into a single call using [`filter_map`].

There are more available adapters. See the list [here][`StreamExt`].

# Implementing `Stream`

The [`Stream`] trait is very similar to the [`Future`] trait.

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Stream {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;

    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}
```

The `Stream::poll_next()` function is much like `Future::poll`, except it can be called
repeatedly to receive many values from the stream. Just as we saw in [Async in
depth][async], when a stream is **not** ready to return a value, `Poll::Pending`
is returned instead. The task's waker is registered. Once the stream should be
polled again, the waker is notified.

The `size_hint()` method is used the same way as it is with [iterators][iter].

Usually, when manually implementing a `Stream`, it is done by composing futures
and other streams. As an example, let's build off of the `Delay` future we
implemented in [Async in depth][async]. We will convert it to a stream that
yields `()` three times at 10 ms intervals

```rust
use tokio_stream::Stream;
# use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;
# use std::time::Instant;

struct Interval {
    rem: usize,
    delay: Delay,
}
# struct Delay { when: Instant }
# impl Future for Delay {
#   type Output = ();
#   fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<()> {
#       Poll::Pending
#   }  
# }

impl Stream for Interval {
    type Item = ();

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<()>>
    {
        if self.rem == 0 {
            // No more delays
            return Poll::Ready(None);
        }

        match Pin::new(&mut self.delay).poll(cx) {
            Poll::Ready(_) => {
                let when = self.delay.when + Duration::from_millis(10);
                self.delay = Delay { when };
                self.rem -= 1;
                Poll::Ready(Some(()))
            }
            Poll::Pending => Poll::Pending,
        }
    }
}
```

## `async-stream`

Manually implementing streams using the [`Stream`] trait can be tedious.
Unfortunately, the Rust programming language does not yet support `async/await`
syntax for defining streams. This is in the works, but not yet ready.

The [`async-stream`] crate is available as a temporary solution. This crate
provides a `stream!` macro that transforms the input into a stream. Using
this crate, the above interval can be implemented like this:

```rust
use async_stream::stream;
# use std::future::Future;
# use std::pin::Pin;
# use std::task::{Context, Poll};
# use tokio_stream::StreamExt;
use std::time::{Duration, Instant};

# struct Delay { when: Instant }
# impl Future for Delay {
#   type Output = ();
#   fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<()> {
#       Poll::Pending
#   }
# }
# async fn dox() {
# let stream =
stream! {
    let mut when = Instant::now();
    for _ in 0..3 {
        let delay = Delay { when };
        delay.await;
        yield ();
        when += Duration::from_millis(10);
    }
}
# ;
# tokio::pin!(stream);
# while let Some(_) = stream.next().await { }
# }
```

[iter]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[`Stream`]: https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html
[`Future`]: https://doc.rust-lang.org/std/future/trait.Future.html
[`StreamExt`]: https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html
[rx]: https://docs.rs/tokio/1/tokio/sync/mpsc/struct.Receiver.html
[`AsyncBufReadExt::lines()`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncBufReadExt.html#method.lines
[next]: https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.next
[`map`]: https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.map
[`take`]: https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.take
[`filter`]: https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter
[`filter_map`]: https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter_map
[pin]: https://doc.rust-lang.org/std/pin/index.html
[async]: async
[`async-stream`]: https://docs.rs/async-stream
[`into_stream()`]: https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Subscriber.html#method.into_stream
[`tokio::pin!`]: https://docs.rs/tokio/1/tokio/macro.pin.html

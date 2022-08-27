---
title: "Channels 消息管道"
---

截至目前我们学了一些Tokio的并发技术, 可以用到这个client例子上。将前面写的server代码移到bin目录:

```text
mkdir src/bin
mv src/main.rs src/bin/server.rs
```

再创建一个新bin文件包含client代码:

```text
touch src/bin/client.rs
```

本页的代码将被写到这个文件中。当你想运行客户端代码前，先在另一个单独的终端窗口
启动server:

```text
cargo run --bin server
```

然后再单独启用client:

```text
cargo run --bin client
```

可以开始写代码了!

即我们想运行两个并发的Redis命令。为每个命令生成一个任务，那么两个命令就并发的执行了。

首先, 我们先尝试如下代码:

```rust,compile_fail
use mini_redis::client;

#[tokio::main]
async fn main() {
    // Establish a connection to the server
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();

    // Spawn two tasks, one gets a key, the other sets a key
    let t1 = tokio::spawn(async {
        let res = client.get("hello").await;
    });

    let t2 = tokio::spawn(async {
        client.set("foo", "bar".into()).await;
    });

    t1.await.unwrap();
    t2.await.unwrap();
}
```

这代码无法编译，因为两个任务都要访问 `client` ，而 `Client` 没有实现 `Copy` 特性, 
在没有支持对象共享的代码前，没法编译。此外，`Client::set` 有 `&mut self` 参数，这
意味这需要独占式的访问它。我们也可以在每个任务中打开一个新连接，但这样的做法不理想。
也不能使用 `std::sync::Mutex` 因为 `.await` 调用导致锁无法释放。可以用 `tokio::sync::Mutex`,
但它只允许一个 in-flight 请求. 如果client 实现了 [pipelining], 异步锁会导致不能充分利用连接能力。

[pipelining]: https://redis.io/topics/pipelining

# 消息传递

正确的做法是使用消息传递。这个模式需要生成一个专用任务来管理 `client` 资源，任何其他
任务想要发出一个请求就发一个消息到 `client` 任务。`client` 任务代表sender发出请求, 
响应也会回传给sender。

使用这个策略，单一连接建立后，管理`client`的任务可以排他的访问 `get` and `set`。
此外, 管道channel相当于一个缓冲。所有操作可以被发送到`client` 任务即使它很忙碌。
一旦 `client` 任务可以处理新请求, 它将从管道中拉取下一个请求。这将得到一个更好的
吞吐能力，可以进一步扩展成连接池。

# Tokio 管道原语

Tokio  [有多种管道][channels], 每一个服务于一个不同的目的。

- [mpsc]: multi-producer, single-consumer channel. 多个值可以被发送。
- [oneshot]: single-producer, single consumer channel. 只能发送单一值。
- [broadcast]: multi-producer, multi-consumer. 多个值可以被发送，每个接收者看到所有值。
- [watch]: single-producer, multi-consumer. 多个值可以被发送，但不保存历史，接收者只看到最新的消息。

如果你需要 multi-producer multi-consumer 管道且每次只有一个接收者能看到一个新消息，
可以使用 [`async-channel`] crate. 也有管道是用在异步代码之外的，如 [`std::sync::mpsc`] 和
[`crossbeam::channel`]. 这些管道会等待消息进而阻塞当前线程，这种事不允许在异步代码中出现。

在这节，我们将使用 [mpsc] 和 [oneshot] 管道。其他管道在后续的章节中涉及。完整代码在 [这里][full].

[channels]: https://docs.rs/tokio/1/tokio/sync/index.html
[mpsc]: https://docs.rs/tokio/1/tokio/sync/mpsc/index.html
[oneshot]: https://docs.rs/tokio/1/tokio/sync/oneshot/index.html
[broadcast]: https://docs.rs/tokio/1/tokio/sync/broadcast/index.html
[watch]: https://docs.rs/tokio/1/tokio/sync/watch/index.html
[`async-channel`]: https://docs.rs/async-channel/
[`std::sync::mpsc`]: https://doc.rust-lang.org/stable/std/sync/mpsc/index.html
[`crossbeam::channel`]: https://docs.rs/crossbeam/latest/crossbeam/channel/index.html

# 自定义消息类型

In most cases, when using message passing, the task receiving the messages
responds to more than one command. In our case, the task will respond to `GET` and
`SET` commands. To model this, we first define a `Command` enum and include a
variant for each command type.

```rust
use bytes::Bytes;

#[derive(Debug)]
enum Command {
    Get {
        key: String,
    },
    Set {
        key: String,
        val: Bytes,
    }
}
```

# Create the channel

In the `main` function, an `mpsc` channel is created.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // Create a new channel with a capacity of at most 32.
    let (tx, mut rx) = mpsc::channel(32);
# tx.send(()).await.unwrap();

    // ... Rest comes here
}
```

The `mpsc` channel is used to **send** commands to the task managing the redis
connection. The multi-producer capability allows messages to be sent from many
tasks. Creating the channel returns two values, a sender and a receiver. The two
handles are used separately. They may be moved to different tasks.

The channel is created with a capacity of 32. If messages are sent faster than
they are received, the channel will store them. Once the 32 messages are stored
in the channel, calling `send(...).await` will go to sleep until a message has
been removed by the receiver.

Sending from multiple tasks is done by **cloning** the `Sender`. For example:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32);
    let tx2 = tx.clone();

    tokio::spawn(async move {
        tx.send("sending from first handle").await;
    });

    tokio::spawn(async move {
        tx2.send("sending from second handle").await;
    });

    while let Some(message) = rx.recv().await {
        println!("GOT = {}", message);
    }
}
```

Both messages are sent to the single `Receiver` handle. It is not possible to
clone the receiver of an `mpsc` channel.

When every `Sender` has gone out of scope or has otherwise been dropped, it is
no longer possible to send more messages into the channel. At this point, the
`recv` call on the `Receiver` will return `None`, which means that all senders
are gone and the channel is closed.

In our case of a task that manages the Redis connection, it knows that it
can close the Redis connection once the channel is closed, as the connection
will not be used again.

# Spawn manager task

Next, spawn a task that processes messages from the channel. First, a client
connection is established to Redis. Then, received commands are issued via the
Redis connection.

```rust
use mini_redis::client;
# enum Command {
#    Get { key: String },
#    Set { key: String, val: bytes::Bytes }
# }
# async fn dox() {
# let (_, mut rx) = tokio::sync::mpsc::channel(10);
// The `move` keyword is used to **move** ownership of `rx` into the task.
let manager = tokio::spawn(async move {
    // Establish a connection to the server
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();

    // Start receiving messages
    while let Some(cmd) = rx.recv().await {
        use Command::*;

        match cmd {
            Get { key } => {
                client.get(&key).await;
            }
            Set { key, val } => {
                client.set(&key, val).await;
            }
        }
    }
});
# }
```

Now, update the two tasks to send commands over the channel instead of issuing
them directly on the Redis connection.

```rust
# #[derive(Debug)]
# enum Command {
#    Get { key: String },
#    Set { key: String, val: bytes::Bytes }
# }
# async fn dox() {
# let (mut tx, _) = tokio::sync::mpsc::channel(10);
// The `Sender` handles are moved into the tasks. As there are two
// tasks, we need a second `Sender`.
let tx2 = tx.clone();

// Spawn two tasks, one gets a key, the other sets a key
let t1 = tokio::spawn(async move {
    let cmd = Command::Get {
        key: "hello".to_string(),
    };

    tx.send(cmd).await.unwrap();
});

let t2 = tokio::spawn(async move {
    let cmd = Command::Set {
        key: "foo".to_string(),
        val: "bar".into(),
    };

    tx2.send(cmd).await.unwrap();
});
# }
````

At the bottom of the `main` function, we `.await` the join handles to ensure the
commands fully complete before the process exits.

```rust
# type Jh = tokio::task::JoinHandle<()>;
# async fn dox(t1: Jh, t2: Jh, manager: Jh) {
t1.await.unwrap();
t2.await.unwrap();
manager.await.unwrap();
# }
```

# Receive responses

The final step is to receive the response back from the manager task. The `GET`
command needs to get the value and the `SET` command needs to know if the
operation completed successfully.

To pass the response, a `oneshot` channel is used. The `oneshot` channel is a
single-producer, single-consumer channel optimized for sending a single value.
In our case, the single value is the response.

Similar to `mpsc`, `oneshot::channel()` returns a sender and receiver handle.

```rust
use tokio::sync::oneshot;

# async fn dox() {
let (tx, rx) = oneshot::channel();
# tx.send(()).unwrap();
# }
```

Unlike `mpsc`, no capacity is specified as the capacity is always one.
Additionally, neither handle can be cloned.

To receive responses from the manager task, before sending a command, a `oneshot`
channel is created. The `Sender` half of the channel is included in the command
to the manager task. The receive half is used to receive the response.

First, update `Command` to include the `Sender`. For convenience, a type alias
is used to reference the `Sender`.

```rust
use tokio::sync::oneshot;
use bytes::Bytes;

/// Multiple different commands are multiplexed over a single channel.
#[derive(Debug)]
enum Command {
    Get {
        key: String,
        resp: Responder<Option<Bytes>>,
    },
    Set {
        key: String,
        val: Bytes,
        resp: Responder<()>,
    },
}

/// Provided by the requester and used by the manager task to send
/// the command response back to the requester.
type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;
```

Now, update the tasks issuing the commands to include the `oneshot::Sender`.

```rust
# use tokio::sync::{oneshot, mpsc};
# use bytes::Bytes;
# #[derive(Debug)]
# enum Command {
#     Get { key: String, resp: Responder<Option<bytes::Bytes>> },
#     Set { key: String, val: Bytes, resp: Responder<()> },
# }
# type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;
# fn dox() {
# let (mut tx, mut rx) = mpsc::channel(10);
# let mut tx2 = tx.clone();
let t1 = tokio::spawn(async move {
    let (resp_tx, resp_rx) = oneshot::channel();
    let cmd = Command::Get {
        key: "hello".to_string(),
        resp: resp_tx,
    };

    // Send the GET request
    tx.send(cmd).await.unwrap();

    // Await the response
    let res = resp_rx.await;
    println!("GOT = {:?}", res);
});

let t2 = tokio::spawn(async move {
    let (resp_tx, resp_rx) = oneshot::channel();
    let cmd = Command::Set {
        key: "foo".to_string(),
        val: "bar".into(),
        resp: resp_tx,
    };

    // Send the SET request
    tx2.send(cmd).await.unwrap();

    // Await the response
    let res = resp_rx.await;
    println!("GOT = {:?}", res);
});
# }
```

Finally, update the manager task to send the response over the `oneshot` channel.

```rust
# use tokio::sync::{oneshot, mpsc};
# use bytes::Bytes;
# #[derive(Debug)]
# enum Command {
#     Get { key: String, resp: Responder<Option<bytes::Bytes>> },
#     Set { key: String, val: Bytes, resp: Responder<()> },
# }
# type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;
# async fn dox(mut client: mini_redis::client::Client) {
# let (_, mut rx) = mpsc::channel::<Command>(10);
while let Some(cmd) = rx.recv().await {
    match cmd {
        Command::Get { key, resp } => {
            let res = client.get(&key).await;
            // Ignore errors
            let _ = resp.send(res);
        }
        Command::Set { key, val, resp } => {
            let res = client.set(&key, val).await;
            // Ignore errors
            let _ = resp.send(res);
        }
    }
}
# }
```

Calling `send` on `oneshot::Sender` completes immediately and does **not**
require an `.await`. This is because `send` on a `oneshot` channel will always
fail or succeed immediately without any form of waiting.

Sending a value on a oneshot channel returns `Err` when the receiver half has
dropped. This indicates the receiver is no longer interested in the response. In
our scenario, the receiver cancelling interest is an acceptable event. The `Err`
returned by `resp.send(...)` does not need to be handled.

You can find the entire code [here][full].

# Backpressure and bounded channels

Whenever concurrency or queuing is introduced, it is important to ensure that the
queueing is bounded and the system will gracefully handle the load. Unbounded queues
will eventually fill up all available memory and cause the system to fail in
unpredictable ways.

Tokio takes care to avoid implicit queuing. A big part of this is the fact that
async operations are lazy. Consider the following:

```rust
# fn async_op() {}
# fn dox() {
loop {
    async_op();
}
# }
# fn main() {}
```

If the asynchronous operation runs eagerly, the loop will repeatedly queue a new
`async_op` to run without ensuring the previous operation completed. This
results in implicit unbounded queuing. Callback based systems and **eager**
future based systems are particularly susceptible to this.

However, with Tokio and asynchronous Rust, the above snippet will **not** result
in `async_op` running at all. This is because `.await` is never called. If the
snippet is updated to use `.await`, then the loop waits for the operation to
complete before starting over.

```rust
# async fn async_op() {}
# async fn dox() {
loop {
    // Will not repeat until `async_op` completes
    async_op().await;
}
# }
# fn main() {}
```

Concurrency and queuing must be explicitly introduced. Ways to do this include:

* `tokio::spawn`
* `select!`
* `join!`
* `mpsc::channel`

When doing so, take care to ensure the total amount of concurrency is bounded. For
example, when writing a TCP accept loop, ensure that the total number of open
sockets is bounded. When using `mpsc::channel`, pick a manageable channel
capacity. Specific bound values will be application specific.

Taking care and picking good bounds is a big part of writing reliable Tokio applications.

[full]: https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs

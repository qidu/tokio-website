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

大多数情况下，使用消息的任务中会响应多种命令。这个例子中任务响应 `GET` 和 `SET` 。
为表达这个，我们定义 `Command` 枚举类，包含每种命令。

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

# 创建管道

在 `main` 函数中创建了 `mpsc` 管道。

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

`mpsc` 管道被用来**send** 命令到管理连接的任务中。它的多生产者能力，允许从多个其他任务中发送消息。
创建管道将返回两个值：发送者sender 和 接收者receiver。这两个消息句柄可独自使用，可能被迁移到不同
任务中。

管道的容量设为32。如果消息发送的快于接收速度，管道将缓存这些消息。一旦保存满32条消息，调用 `send(...).await`
将导致发送者所在的线程进入休眠，直到有一个消息从管道中被接收者取走。

通过克隆**cloning** `Sender`句柄实现从多个任务中发送消息。例如:

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

多个来源消息都被发送到同一个接收者 `Receiver` 句柄。在 `mpsc` 管道上不能克隆接收者。

每个 `Sender` 超出生命周期时被释放，将不在能用它发送消息到管道。此时，在接收者上调用`recv`
将返回 `None`，它意味着所有的发送者都已被销毁，管道已关闭。

在我们的例子中，管理Redis连接的任务相应知道在管道管理时也该关闭连接。

# 生成管理任务

接着，生成任务来管理消息。首先，建立一个连接到Redis；然后，收到命令后再被发送到Redis连接上。

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

现在可更新2个任务，通过管道发送命令，代替直接发送命令到Redis连接上。

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

在 `main` 函数的结尾，我们调用join句柄的 `.await` 以确保命令在进程退出前都被完全处理好。

```rust
# type Jh = tokio::task::JoinHandle<()>;
# async fn dox(t1: Jh, t2: Jh, manager: Jh) {
t1.await.unwrap();
t2.await.unwrap();
manager.await.unwrap();
# }
```

# 接收响应

最后一步是从管理任务收回响应。`GET`命令需要得到返回值，而`SET` 需要知道操作结果是否成功。

为了传递结果，使用 `oneshot` 管道。`oneshot` 是优化了以发送单一值的single-producer, single-consumer管道。
在我们的例子里，这个单一值就是响应结果。

与 `mpsc`相似，调用`oneshot::channel()` 返回一对句柄。

```rust
use tokio::sync::oneshot;

# async fn dox() {
let (tx, rx) = oneshot::channel();
# tx.send(()).unwrap();
# }
```

与 `mpsc`不同，无需指定管道容量因为它始终是一。此外，任何一个句柄都不能被克隆。

为从连接管理任务接收响应，在发送命令前，需要创建一个 `oneshot`管道。其中`Sender` 句柄
需要包含在发送给管理任务的命令中，而接收句柄留着等待响应。
首先，更新 `Command` 以包含 `Sender`。为方便，定义一个类型别名来引用`Sender`。

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

现在更新这个任务，发出包含也有 `oneshot::Sender`的任务。

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

最后，更新连接管理任务，以通过 `oneshot` 管道发送响应。

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

调用`oneshot::Sender`的`send`方法会立即完成，不需要 `.await`。这是因为`oneshot` 
管道的`send`函数将总是返回失败或立即成功而不会有任何形式的等待。

在oneshot管道上发送消息只在接收者被销毁时才得到`Err`返回值。这意味着接收者对响应值不
再感兴趣。在我们的场景里，接收者取消等待返回值也是可接受的事件。调用`resp.send(...)`的`Err`
返回值不需要处理。

你可以 [在这里][full]找到完整代码。

# 受限管道上的背压

任何时候引入并发和消息队列时，确保队列长度是受限的就很重要。系统可以平滑负载。而不受限的消息队列
会最终填满耗尽全部内存，引起系统掉进不可预知的运行状态。

Tokio小心避免显示的消息队列，一个重要的原因是由于异步操作全是lazy惰性的。考虑如下代码：

```rust
# fn async_op() {}
# fn dox() {
loop {
    async_op();
}
# }
# fn main() {}
```

如果异步操作急切执行，loop循环将重复加入一个新的`async_op`执行而不会等待前一个结果。这导致
一个隐式的不受限队列。基于回调的系统，和急切的**eager**延后执行系统，对此都很敏感。

但是，在Tokio和Rust异步编程中，上面的代码并不会 **not** 导致`async_op`执行。这是因为 `.await` 
还没被调用。如果更新一下调用 `.await`，这样loop循环将等待操作完成后然后才进入下一个执行。

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

并发和队列需要显示引入。这样做的方法包括:

* `tokio::spawn`
* `select!`
* `join!`
* `mpsc::channel`

当这么做时，当心确保全部并发任务的数量是受限的。例如，当编写TCP连接接收循环时，确保打开的全部
连接数量是受限的。当使用`mpsc::channel`管道，选择一个可容忍的容量初始化值。指定容量会使程序
运行更有确定。

小心选择合适的受限容量，是写好可靠的Tokio应用的重要部分。

[full]: https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs

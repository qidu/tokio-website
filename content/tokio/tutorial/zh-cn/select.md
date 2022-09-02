---
title: "Select"
---

截至目前，当我们想给程序增加并发，我们会生成一个新任务。现在进一步覆盖一些额外的方法，
以通过Tokio执行异步代码。

# `tokio::select!`

宏 `tokio::select!` 允许等待在多个异步计算上，并能在任意**一个**异步计算完成时返回。

例如:

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    tokio::spawn(async {
        let _ = tx1.send("one");
    });

    tokio::spawn(async {
        let _ = tx2.send("two");
    });

    tokio::select! {
        val = rx1 => {
            println!("rx1 completed first with {:?}", val);
        }
        val = rx2 => {
            println!("rx2 completed first with {:?}", val);
        }
    }
}
```

使用了两个 oneshot 管道。任何一个管道有可能先收到消息。`select!` 语句 await 在两个管道上，
并将 `val` 绑定到任务返回值上。当 `tx1` 和 `tx2` 中任何一个就绪，响应的代码块就会先执行。

而**未完成** 的分支将会被丢弃。在这个例子，计算任务会等待在每个管道的 `oneshot::Receiver` 上。
没完成 `oneshot::Receiver` 对应的管道将被丢弃。

## 取消

使用Rust异步编程，通过丢弃一个future来取消任务。从["深入异步"][async]中回忆，异步操作是
通过lazy的future来实现的。这些操作只会在future被poll时执行。如果要丢弃future，对应的操作
就不会再处理了因为所有关联的状态也都被丢弃了。

也就是说，有时一个异步操作会生成新任务或启动它将在后台运行的任务。例如，前面例子中，一个任务
生成了用于回送消息。通常，任务将执行一些计算并生成返回值。

Futures 或其他类型能实现 `Drop` 特性以清理后台资源。Tokio的 `oneshot::Receiver` 实现了 `Drop` 
以发送关闭通知给对端 `Sender` 。管道发送端能收到关闭通知并取消释放正处理中的消息操作。


```rust
use tokio::sync::oneshot;

async fn some_operation() -> String {
    // Compute value here
# "wut".to_string()
}

#[tokio::main]
async fn main() {
    let (mut tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    tokio::spawn(async {
        // Select on the operation and the oneshot's
        // `closed()` notification.
        tokio::select! {
            val = some_operation() => {
                let _ = tx1.send(val);
            }
            _ = tx1.closed() => {
                // `some_operation()` is canceled, the
                // task completes and `tx1` is dropped.
            }
        }
    });

    tokio::spawn(async {
        let _ = tx2.send("two");
    });

    tokio::select! {
        val = rx1 => {
            println!("rx1 completed first with {:?}", val);
        }
        val = rx2 => {
            println!("rx2 completed first with {:?}", val);
        }
    }
}
```

[async]: async

## `Future` 实现

为更好的理解 `select!` 如何工作，我们先看看一个假设的 `Future` 实现是什么样的。这是简化版。
实现上，`select!` 包括额外的功能，如随机选择异步分支任务以先poll。

```rust
use tokio::sync::oneshot;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct MySelect {
    rx1: oneshot::Receiver<&'static str>,
    rx2: oneshot::Receiver<&'static str>,
}

impl Future for MySelect {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if let Poll::Ready(val) = Pin::new(&mut self.rx1).poll(cx) {
            println!("rx1 completed first with {:?}", val);
            return Poll::Ready(());
        }

        if let Poll::Ready(val) = Pin::new(&mut self.rx2).poll(cx) {
            println!("rx2 completed first with {:?}", val);
            return Poll::Ready(());
        }

        Poll::Pending
    }
}

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    // use tx1 and tx2
# tx1.send("one").unwrap();
# tx2.send("two").unwrap();

    MySelect {
        rx1,
        rx2,
    }.await;
}
```

这里 `MySelect` future 包含对应每个分支的future。当 `MySelect` 被poll了，第一个
分支也会被poll。如果它完成了，就会使用它的返回值完成 `MySelect` 的poll过程。在 `.await` 
收到future的输出，就会丢掉它。这导致两个分支都会被丢弃。因为另一个分支并没完成，
对应的操作就被有效的取消了。

从前面的章节记住如下:

> 当一个future返回 `Poll::Pending`状态，它**必须** 确保相应waker在后面某个时间点被通知。
> 忘记这么做会导致任务被无限挂起。

这里在 `MySelect`的实现中没有显示使用 `Context` 参数，
代替方法是，waker 的要求是通过传递 `cx` 内部future。内部 future 必须也满足 waker 要求，by only
收到内部future的 `Poll::Pending`返回值时也应返回 `Poll::Pending` ，`MySelect` 也就符合
waker的要求。

# 语法

宏 `select!` 可以处理多于两个分支。目前的最大闲置是 64 个分支。每个分支的格式是:

```text
<pattern> = <async expression> => <handler>,
```

当 `select` 宏被解释后，所有的 `<async expression>` 将被聚合以并发执行。当其中一个异步
分支执行完，返回值将会与 `<pattern>` 进行匹配。如果结果符合模式，那么所有剩余的异步分支
将被丢弃，匹配上的分支对应的 `<handler>` 将被执行。`<handler>` 分支将访问 `<pattern>`建立的任何绑定。

最基本的 `<pattern>` 例子是变量名，异步表达式的返回值将被绑定到该变量上，`<handler>` 以
该变量名来访问返回值。这也是为什么最前面的例子中把 `val` 用在 `<pattern>` 上以便 `<handler>` 
能访问它。

如果 `<pattern>` **不匹配** 异步表达式返回值，那么剩余的异步表达式将继续并发执行直到下一个完成。
那时，同样的逻辑会用在新返回值上。

由于 `select!` 能使用任何异步表达式，可能定义更复杂的执行过程用来 select 它。

这里我们select在 `oneshot` 管道和一个TCP连接上。

```rust
use tokio::net::TcpStream;
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    // Spawn a task that sends a message over the oneshot
    tokio::spawn(async move {
        tx.send("done").unwrap();
    });

    tokio::select! {
        socket = TcpStream::connect("localhost:3465") => {
            println!("Socket connected {:?}", socket);
        }
        msg = rx => {
            println!("received message first {:?}", msg);
        }
    }
}
```

这里我们select在oneshot管道和`TcpListener`的socket接收上。

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        tx.send(()).unwrap();
    });

    let mut listener = TcpListener::bind("localhost:3465").await?;

    tokio::select! {
        _ = async {
            loop {
                let (socket, _) = listener.accept().await?;
                tokio::spawn(async move { process(socket) });
            }

            // Help the rust type inferencer out
            Ok::<_, io::Error>(())
        } => {}
        _ = rx => {
            println!("terminating accept loop");
        }
    }

    Ok(())
}
# async fn process(_: tokio::net::TcpStream) {}
```

这个accept循环运行直到遇到错误或者管道 `rx` 端收到消息。这里模式 `_` 表示我们对该异步计算的返回值没有兴趣。

# 返回值

宏 `tokio::select!` 返回解释后的 `<handler>` 表达式的值。 

```rust
async fn computation1() -> String {
    // .. computation
# unimplemented!();
}

async fn computation2() -> String {
    // .. computation
# unimplemented!();
}

# fn dox() {
#[tokio::main]
async fn main() {
    let out = tokio::select! {
        res1 = computation1() => res1,
        res2 = computation2() => res2,
    };

    println!("Got = {}", out);
}
# }
```

因此，要求**每一个**分支对应的 `<handler>` 表达式应返回相同类型的值。如果不需要
`select!` 表达式的返回值，那么返回空类型 `()` 是好的实践。

# 错误

使用操作符 `?` 操作符从表达式中传播了错误。如何实现这个取决于`?`是否被一部表达式或处理函数使用。
在异步表达式中使用 `?` 可将错误传播出该表达式。这使得异步表达式输出一个 `Result`。在句柄中
使用 `?` 可立即将错误传播出 `select!` 表达式。
让我们再看看accept循环的例子:

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    // [setup `rx` oneshot channel]
# let (tx, rx) = oneshot::channel();
# tx.send(()).unwrap();

    let listener = TcpListener::bind("localhost:3465").await?;

    tokio::select! {
        res = async {
            loop {
                let (socket, _) = listener.accept().await?;
                tokio::spawn(async move { process(socket) });
            }

            // Help the rust type inferencer out
            Ok::<_, io::Error>(())
        } => {
            res?;
        }
        _ = rx => {
            println!("terminating accept loop");
        }
    }

    Ok(())
}
# async fn process(_: tokio::net::TcpStream) {}
```

注意 `listener.accept().await?`。操作符 `?` 将错误传播出这表达式，传到 `res` 捆绑变量。
在出错时， `res` 将被设置为 `Err(_)`。然后，在处理函数中，操作符 `?` 再次被使用。语句 `res?`
将把错误传播出 `main` 函数。

# 模式匹配

回忆宏 `select!` 分支语法的定义:

```text
<pattern> = <async expression> => <handler>,
<模式变量> = <异步表达式> => <处理函数>,
```

到目前为止，我们只是为 `<pattern>` 使用过捆绑变量。尽管如此，任意Rust模式都能被使用。
例如，假设我们收到多个管道的消息，或许执行如下代码:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (mut tx1, mut rx1) = mpsc::channel(128);
    let (mut tx2, mut rx2) = mpsc::channel(128);

    tokio::spawn(async move {
        // Do something w/ `tx1` and `tx2`
# tx1.send(1).await.unwrap();
# tx2.send(2).await.unwrap();
    });

    tokio::select! {
        Some(v) = rx1.recv() => {
            println!("Got {:?} from rx1", v);
        }
        Some(v) = rx2.recv() => {
            println!("Got {:?} from rx2", v);
        }
        else => {
            println!("Both channels closed");
        }
    }
}
```

在这个例子中，宏 `select!` 表达式等待从 `rx1` 和 `rx2` 上收取消息。如果一个管道关闭了，
等待的 `recv()` 调用将返回 `None`。这**并不** 匹配模式，该分支将不可用。宏 `select!` 
表达式将继续等待剩下的分析。

注意这里 `select!` 表达式包含一个 `else` 分支。 `select!` 表达式必须计算一个返回值，
当使用模式匹配时，也可能**没有**任何分支匹配关联的模式。如果出现这样的情况，`else` 分支
将被评估计算。

# 借用

当生成任务时，生成的异步表达式必须拥有所有关联数据。宏 `select!` 没有这个限制。
每个分支的异步表达式可以借用数据，并发操作。根据Rust的借用规则，多个异步表达式
可以不可变的借用单个值**或** 仅单个异步表达式可以可变的借用一个值。

让我们看看一些例子。这里，我们同时发送相同的数据到两个不同的TCP连接目标。

```rust
use tokio::io::AsyncWriteExt;
use tokio::net::TcpStream;
use std::io;
use std::net::SocketAddr;

async fn race(
    data: &[u8],
    addr1: SocketAddr,
    addr2: SocketAddr
) -> io::Result<()> {
    tokio::select! {
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr1).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr2).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        else => {}
    };

    Ok(())
}
# fn main() {}
```

这个 `data` 变量被两个异步表达式**不可变**的借用了。当其中一个操作成功，另一个分支被丢弃
由于要匹配的模式是 `Ok(_)`，如果一个表达式没匹配上，另一个将继续执行。

当执行每个分支的 `<handler>` 时，宏 `select!` 确保只有一个 `<handler>` 会运行。因此，
每个 `<handler>` 可以可变的借用同样的变量数据。

例如这里就可以修改两个处理程序中的 `out` 变量（可变借用）:

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    let mut out = String::new();

    tokio::spawn(async move {
        // Send values on `tx1` and `tx2`.
# let _ = tx1.send("one");
# let _ = tx2.send("two");
    });

    tokio::select! {
        _ = rx1 => {
            out.push_str("rx1 completed");
        }
        _ = rx2 => {
            out.push_str("rx2 completed");
        }
    }

    println!("{}", out);
}
```

# 循环

宏 `select!` 经常用在循环中。本节将通过一些例子以展示在循环中使用宏`select!`的常见方案。
我们从select多个管道开始:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx1, mut rx1) = mpsc::channel(128);
    let (tx2, mut rx2) = mpsc::channel(128);
    let (tx3, mut rx3) = mpsc::channel(128);
# tx1.clone().send("hello").await.unwrap();
# drop((tx1, tx2, tx3));

    loop {
        let msg = tokio::select! {
            Some(msg) = rx1.recv() => msg,
            Some(msg) = rx2.recv() => msg,
            Some(msg) = rx3.recv() => msg,
            else => { break }
        };

        println!("Got {:?}", msg);
    }

    println!("All channels have been closed.");
}
```

这个例子selects了三个管道接收端。当任何一个收到一条消息时，它将被写道STDOUT。
当一个管道关闭，`recv()` 会返回 `None`。使用模式匹配时，宏 `select!` 将继续
等待其他剩余的管道。当所有的管道都关闭了， `else` 分支将被执行而loop会break终止。

宏 `select!` 随机的挑选分支来执行读取操作。当多个管道都有值可读时，随机选一个
管道来接收消息。这可以处理一些当接收消息慢于发送消息意味着管道将可能填满的情况。
如果 `select!` **不**随机选择分支先执行检查，则每次循环时，总先选择`rx1`检查。
而如果`rx1`总是有消息可处理，则剩下的管道将永远不会被检查执行到。
> 提示：
> 如果当 `select!` 执行，多个管道有缓存住的消息，只有一个管道中的消息会弹出。
> 其他管道保持为没处理，它们的消息继续保留在管道中，直到下一个循环随机选择到。
> 没有消息会丢失。

## Resuming an async operation

Now we will show how to run an asynchronous operation across multiple calls to
`select!`. In this example, we have an MPSC channel with item type `i32`, and an
asynchronous function. We want to run the asynchronous function until it
completes or an even integer is received on the channel.

```rust
async fn action() {
    // Some asynchronous logic
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);    
#   tokio::spawn(async move {
#       let _ = tx.send(1).await;
#       let _ = tx.send(2).await;
#   });
    
    let operation = action();
    tokio::pin!(operation);
    
    loop {
        tokio::select! {
            _ = &mut operation => break,
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    break;
                }
            }
        }
    }
}
```

Note how, instead of calling `action()` in the `select!` macro, it is called
**outside** the loop. The return of `action()` is assigned to `operation`
**without** calling `.await`. Then we call `tokio::pin!` on `operation`.

Inside the `select!` loop, instead of passing in `operation`, we pass in `&mut
operation`. The `operation` variable is tracking the in-flight asynchronous
operation. Each iteration of the loop uses the same operation instead of issuing
a new call to `action()`.

The other `select!` branch receives a message from the channel. If the message
is even, we are done looping. Otherwise, start the `select!` again.

This is the first time we use `tokio::pin!`. We aren't going to get into the
details of pinning yet. The thing to note is that, to `.await` a reference,
the value being referenced must be pinned or implement `Unpin`.

If we remove the `tokio::pin!` line and try to compile, we get the following
error:

```text
error[E0599]: no method named `poll` found for struct
     `std::pin::Pin<&mut &mut impl std::future::Future>`
     in the current scope
  --> src/main.rs:16:9
   |
16 | /         tokio::select! {
17 | |             _ = &mut operation => break,
18 | |             Some(v) = rx.recv() => {
19 | |                 if v % 2 == 0 {
...  |
22 | |             }
23 | |         }
   | |_________^ method not found in
   |             `std::pin::Pin<&mut &mut impl std::future::Future>`
   |
   = note: the method `poll` exists but the following trait bounds
            were not satisfied:
           `impl std::future::Future: std::marker::Unpin`
           which is required by
           `&mut impl std::future::Future: std::future::Future`
```

Although we covered `Future` in [the previous chapter][async], this error still isn't
very clear. If you hit such an error about `Future` not being implemented when attempting
to call `.await` on a **reference**, then the future probably needs to be pinned.

Read more about [`Pin`][pin] on the [standard library][pin].

[pin]: https://doc.rust-lang.org/std/pin/index.html

## Modifying a branch

Let's look at a slightly more complicated loop. We have:

1. A channel of `i32` values.
2. An async operation to perform on `i32` values.

The logic we want to implement is:

1. Wait for an **even** number on the channel.
2. Start the asynchronous operation using the even number as input.
3. Wait for the operation, but at the same time listen for more even numbers on
   the channel.
4. If a new even number is received before the existing operation completes,
   abort the existing operation and start it over with the new even number.

```rust
async fn action(input: Option<i32>) -> Option<String> {
    // If the input is `None`, return `None`.
    // This could also be written as `let i = input?;`
    let i = match input {
        Some(input) => input,
        None => return None,
    };
    // async logic here
#   Some(i.to_string())
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);
    
    let mut done = false;
    let operation = action(None);
    tokio::pin!(operation);
    
    tokio::spawn(async move {
        let _ = tx.send(1).await;
        let _ = tx.send(3).await;
        let _ = tx.send(2).await;
    });
    
    loop {
        tokio::select! {
            res = &mut operation, if !done => {
                done = true;

                if let Some(v) = res {
                    println!("GOT = {}", v);
                    return;
                }
            }
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    // `.set` is a method on `Pin`.
                    operation.set(action(Some(v)));
                    done = false;
                }
            }
        }
    }
}
```

We use a similar strategy as the previous example. The async fn is called
outside of the loop and assigned to `operation`. The `operation` variable is
pinned. The loop selects on both `operation` and the channel receiver.

Notice how `action` takes `Option<i32>` as an argument. Before we receive the
first even number, we need to instantiate `operation` to something. We make
`action` take `Option` and return `Option`. If `None` is passed in, `None` is
returned. The first loop iteration, `operation` completes immediately with
`None`.

This example uses some new syntax. The first branch includes `, if !done`. This
is a branch precondition. Before explaining how it works, let's look at what
happens if the precondition is omitted. Leaving out `, if !done` and running the
example results in the following output:

```text
thread 'main' panicked at '`async fn` resumed after completion', src/main.rs:1:55
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

This error happens when attempting to use `operation` **after** it has already
completed. Usually, when using `.await`, the value being awaited is consumed. In
this example, we await on a reference. This means `operation` is still around
after it has completed.

To avoid this panic, we must take care to disable the first branch if
`operation` has completed. The `done` variable is used to track whether or not
`operation` completed. A `select!` branch may include a **precondition**. This
precondition is checked **before** `select!` awaits on the branch. If the
condition evaluates to `false` then the branch is disabled. The `done` variable
is initialized to `false`. When `operation` completes, `done` is set to `true`.
The next loop iteration will disable the `operation` branch. When an even
message is received from the channel, `operation` is reset and `done` is set to
`false`.

# Per-task concurrency

Both `tokio::spawn` and `select!` enable running concurrent asynchronous
operations. However, the strategy used to run concurrent operations differs. The
`tokio::spawn` function takes an asynchronous operation and spawns a new task to
run it. A task is the object that the Tokio runtime schedules. Two different
tasks are scheduled independently by Tokio. They may run simultaneously on
different operating system threads. Because of this, a spawned task has the same
restriction as a spawned thread: no borrowing.

The `select!` macro runs all branches concurrently **on the same task**. Because
all branches of the `select!` macro are executed on the same task, they will
never run **simultaneously**. The `select!` macro multiplexes asynchronous
operations on a single task.

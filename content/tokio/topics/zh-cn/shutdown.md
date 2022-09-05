---
title: "优雅关闭"
---

这页文章的主要目的是，提供一个关于异步程序如何合适地实现退出关闭程序的概览。

有三部分来实现优雅关闭:

 * 搞清楚何时关闭.
 * 通知程序的每个模块关闭.
 * 等待其他部分完成关闭.

本文剩下部分将探讨这三部分。一个真实的关闭方法实现可以在[mini-redis]看到。
具体在 [`src/server.rs`][server.rs] 和 [`src/shutdown.rs`][shutdown.rs] 两个文件中。

## 搞清楚何时关闭

这当然取决于该应用本身。一个普遍的关闭标准是应用收到了操作系统的信号。通常例如当你在终端
窗口传递ctrl+c给运行中的程序。为检测到这信号，Tokio提供了 [`tokio::signal::ctrl_c`][ctrl_c]
函数，它将睡眠到有信号收到。你可以这样使用它:
```rs
use tokio::signal;

#[tokio::main]
async fn main() {
    // ... spawn application as separate task ...

    match signal::ctrl_c().await {
        Ok(()) => {},
        Err(err) => {
            eprintln!("Unable to listen for shutdown signal: {}", err);
            // we also shut down in case of error
        },
    }

    // send shutdown signal to application and wait
}
```
如果你有多个不同的关闭条件，可以使用 [an mpsc channel] 管道来发送关闭信号到一个特定的地方。然后
[select] 在 [`ctrl_c`][ctrl_c] 和该管道上。例如：

```rs
use tokio::signal;
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (shutdown_send, shutdown_recv) = mpsc::unbounded_channel();

    // ... spawn application as separate task ...
    //
    // application uses shutdown_send in case a shutdown was issued from inside
    // the application

    tokio::select! {
        _ = signal::ctrl_c() => {},
        _ = shutdown_recv.recv() => {},
    }

    // send shutdown signal to application and wait
}
```

## 通知其他模块关闭

最常见用来告诉应用的每个部分关闭的工具是 [broadcast channel][broadcast]。它很简单：
应用中的每个任务都有一个广播管道的接收端，当有广播消息时，每个任务关闭自己。通常用
[`tokio::select`][select]接收广播消息。例如，在 mini-redis 里每个任务这样接收
关闭消息：
```rs
let next_frame = tokio::select! {
    res = self.connection.read_frame() => res?,
    _ = self.shutdown.recv() => {
        // If a shutdown signal is received, return from `run`.
        // This will result in the task terminating.
        return Ok(());
    }
};
```
在 mini-redis 里，任务在收到关闭消息时立刻退出，但在有些情况下，你将在实际退出前执行一个关闭过程。
例如，在有些情况，你在退出前需要把数据刷入文件或数据库。如果任务还关联一个连接，你也许要发送关闭信息
到连接上。

把广播管道包装在一个struct上通常是个好主意。在 [这里][shutdown.rs] 可以找到例子。

值得一提的是，你可以用 [watch channel][watch] 实现相同功能。这两个选择没有特别大的区别。

## 等待其他模块完成关闭

一旦你通知了其他模块关闭，你需要等待它们完成关闭。最简单的方式是使用 [an mpsc channel]。代替回传通知，
你只需要等待在这个管道等它关闭，它在发送端丢弃时关闭。

作为这个模式的简单例子，下面的例子中将生成10个任务，然后使用mpsc管道来等待它们关闭。
```rs
use tokio::sync::mpsc::{channel, Sender};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (send, mut recv) = channel(1);

    for i in 0..10 {
        tokio::spawn(some_operation(i, send.clone()));
    }

    // Wait for the tasks to finish.
    //
    // We drop our sender first because the recv() call otherwise
    // sleeps forever.
    drop(send);

    // When every sender has gone out of scope, the recv call
    // will return with an error. We ignore the error.
    let _ = recv.recv().await;
}

async fn some_operation(i: u64, _sender: Sender<()>) {
    sleep(Duration::from_millis(100 * i)).await;
    println!("Task {} shutting down.", i);

    // sender goes out of scope ...
}
```
一个非常重要的细节是，等待关闭的任务通常持有管道发送端。这种情况，你必须确保在等待管道关闭前
丢弃发送端。

[ctrl_c]: https://docs.rs/tokio/1/tokio/signal/fn.ctrl_c.html
[an mpsc channel]: https://docs.rs/tokio/1/tokio/sync/mpsc/index.html
[select]: https://docs.rs/tokio/1/tokio/macro.select.html
[broadcast]: https://docs.rs/tokio/1/tokio/sync/broadcast/index.html
[watch]: https://docs.rs/tokio/1/tokio/sync/watch/index.html
[shutdown.rs]: https://github.com/tokio-rs/mini-redis/blob/master/src/shutdown.rs
[server.rs]: https://github.com/tokio-rs/mini-redis/blob/master/src/server.rs
[mini-redis]: https://github.com/tokio-rs/mini-redis/

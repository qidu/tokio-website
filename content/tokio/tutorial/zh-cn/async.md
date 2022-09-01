---
title: "深入异步"
---

截至目前，我们已经完成了对Rust和Tokio非常综合的案例学习。现在我们将更深入到Rust的异步编程
运行时模型。
在本指南的开头处，我们提到Rust异步编程采用了一个独特的办法。现在我们来解释为什么独特。

# Futures 延迟执行 （承诺、约定、期约）

快速过一下基本异步函数，它没有比指南中学过的例子更复杂。

```rust
use tokio::net::TcpStream;

async fn my_async_fn() {
    println!("hello from async");
    let _socket = TcpStream::connect("127.0.0.1:3000").await.unwrap();
    println!("async TCP operation complete");
}
```

我们调用该函数它返回一些值。我们在该值上继续调用`.await`。

```rust
# async fn my_async_fn() {}
#[tokio::main]
async fn main() {
    let what_is_this = my_async_fn();
    // Nothing has been printed yet.
    what_is_this.await;
    // Text has been printed and socket has been
    // established and closed.
}
```

函数`my_async_fn()`的返回值就是一个future。future是一个实现了[`std::future::Future`][trait] trait 的值，
它由标准库提供。它包含了待处理的异步计算的返回值。

标准库里 [`std::future::Future`][trait] trait 定义如下:

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Self::Output>;
}
```

[关联类型][assoc] `Output` 返回值是由future在完成后产生的。[`Pin`][pin] 类型是Rust如何
支持在`async` 函数中借用变量的基础。可以从 [标准库][pin] 文档中参考更多细节。

不像其他语言中的futures实现，Rust的future并不表示一个正在背后运行的计算过程，相反Rust的future
只表示计算过程本身。future的所有者负责通过调用`Future::poll`操作来执行这个计算过程。

## 实现 `Future`

我们来实现一个简单的future:

1. 等待到指定时刻
2. 输出一些文本到STDOUT
3. 产出一个字符串

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // 暂时忽略这行.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    let out = future.await;
    assert_eq!(out, "done");
}
```

## Async fn 作为 Future

在main函数中，我们实例化了future 并调用 `.await`。在异步函数里，我们可以在任何实现过`Future`的值上
调用 `.await`。相应的，调用`async` 函数返回实现了`Future`的匿名类型值。而调用 `async fn main()`，
大体生成如下future:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum MainFuture {
    // Initialized, never polled
    State0,
    // Waiting on `Delay`, i.e. the `future.await` line.
    State1(Delay),
    // The future has completed.
    Terminated,
}
# struct Delay { when: Instant };
# impl Future for Delay {
#     type Output = &'static str;
#     fn poll(self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<&'static str> {
#         unimplemented!();
#     }
# }

impl Future for MainFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<()>
    {
        use MainFuture::*;

        loop {
            match *self {
                State0 => {
                    let when = Instant::now() +
                        Duration::from_millis(10);
                    let future = Delay { when };
                    *self = State1(future);
                }
                State1(ref mut my_future) => {
                    match Pin::new(my_future).poll(cx) {
                        Poll::Ready(out) => {
                            assert_eq!(out, "done");
                            *self = Terminated;
                            return Poll::Ready(());
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                Terminated => {
                    panic!("future polled after completion")
                }
            }
        }
    }
}
```

Rust futures 是状态机 **state machines**。这里 `MainFuture` 被表达为future可能状态的`enum`类型。
此future从状态 `State0` 开始。当调用其 `poll` 方法时，future尝试尽可能迁移其内部状态。如果feature
能执行完，就将这次异步执行的结果封装在状态 `Poll::Ready` 里一起返回。

如果该future还无法完成，通常因为还在等待某些未就绪的资源，那么返回 `Poll::Pending` 。当收到
`Poll::Pending` 返回值时，表示该future将会在以后某个时间点完成，调用者需要在后面继续调用 `poll`。

我们也需要注意有些futures会组合其他futures。调用外层future的`poll`会导致进一步调用到内层future的相应函数。

# Executors 执行器

异步的Rust函数返回futures。Futures 的`poll` 函数必须被调用以转换它们的状态。Futures可以由其他futures组合而成。
所以问题是，是什么在调用最外层future的 `poll` 函数？

回忆前面，为执行异步函数，它们需要被传递给 `tokio::spawn` 或者将main函数注释为`#[tokio::main]`。
这导致将外层future提交给Tokio执行器。执行器负责调用外层 `Future::poll` 驱动异步函数过程执行到完成状态。

## Mini Tokio

为了更好理解这些完整过程，我们先实现一个最小化的Tokio。完整代码在[这里][mini-tokio].

```rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use futures::task;

fn main() {
    let mut mini_tokio = MiniTokio::new();

    mini_tokio.spawn(async {
        let when = Instant::now() + Duration::from_millis(10);
        let future = Delay { when };

        let out = future.await;
        assert_eq!(out, "done");
    });

    mini_tokio.run();
}
# struct Delay { when: Instant }
# impl Future for Delay {
#     type Output = &'static str;
#     fn poll(self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<&'static str> {
#         Poll::Ready("done")
#     }
# }

struct MiniTokio {
    tasks: VecDeque<Task>,
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new(),
        }
    }
    
    /// Spawn a future onto the mini-tokio instance.
    fn spawn<F>(&mut self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        self.tasks.push_back(Box::pin(future));
    }
    
    fn run(&mut self) {
        let waker = task::noop_waker();
        let mut cx = Context::from_waker(&waker);
        
        while let Some(mut task) = self.tasks.pop_front() {
            if task.as_mut().poll(&mut cx).is_pending() {
                self.tasks.push_back(task);
            }
        }
    }
}
```

这里执行了异步代码。一个 `Delay` 实例随指定延后时间被生成，并调用它的await。尽管这样，截至目前我们
的实现有一个主要**缺陷**。执行器从不进入休眠。执行器持续遍历全部被生成的futures并用poll调用它们。
大多数时候，futures 并不会是就绪状态等着进一步执行，而是会一再返回 `Poll::Pending` 。这个进程
会消耗CPU周期，并不是太有效率。

理想情况，我们希望 mini-tokio 只在 future 能被改变状态时去 poll 它。这会在对应阻塞等待的资源就绪
能进一步操作时才出现。例如该任务实现要从TCP socket读取数据，我们只会在TCP socket上收到数据时poll这个任务。
在我们的例子中，这个任务要阻塞到指定的 `Instant` 时刻到达时才完成。相应的，mini-tokio 只要在时刻到时 poll该任务。

为实现这个过程，当一个资源被poll了但其状态**还没有** 就绪，这资源应在它的状态迁移到就绪时，立刻发出一个唤醒通知。

# Wakers 唤醒者

所以缺的就是Wakers。这就是在等待的资源就绪时，能够通知对应等待中的任务的机制。 

让我们再看看 `Future::poll` 的定义:

```rust,compile_fail
fn poll(self: Pin<&mut Self>, cx: &mut Context)
    -> Poll<Self::Output>;
```

`Context` 的`poll` 参数有 `waker()` 方法。这个方法会返回一个[`Waker`] 约束在当前任务上. [`Waker`] 有一个 `wake()` 方法。
调用这个方法，会传递信号告诉执行器：与它关联的任务可以被再次调度执行了。资源绪时调用 `wake()` 以通知执行器此时poll这任务可以
迁移到新状态。

## 更新 `Delay`

我们可以用waker更新 `Delay`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use std::thread;

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Get a handle to the waker for the current task
            let waker = cx.waker().clone();
            let when = self.when;

            // Spawn a timer thread.
            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                waker.wake();
            });

            Poll::Pending
        }
    }
}
```

现在，一旦指定的时间过去，调用中的任务被通知，执行器会确保该任务再次被调度执行。下一步是
更新 mini-tokio 去监听唤醒通知。

我们 `Delay` 实现仍有一些遗留问题，将在后面再修复它们。

[[警告]]
| 当future 返回 `Poll::Pending`，它**必须**确定对应的waker会再某个时间点被唤醒。 
| 忘记这个步骤会导致任务被无限期挂起。
|
| 在返回 `Poll::Pending` 后忘记唤醒任务是比较常见的出错原因。

回忆首次迭代 `Delay`。 以下是 future 实现:

```rust
# use std::future::Future;
# use std::pin::Pin;
# use std::task::{Context, Poll};
# use std::time::Instant;
# struct Delay { when: Instant }
impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

在返回 `Poll::Pending` 前，我们调用了 `cx.waker().wake_by_ref()`。这就满足了future约定。
因为返回 `Poll::Pending`，我们有责任给waker以信号。由于没有额外的timer线程，我们直接在本线程
内部唤醒了waker。这么做会导致future被立刻重调度执行。很可能它依赖的资源还是没有准备好。

注意，你可以被允许比实际需要更频繁的唤醒waker。在有些情况下，我们甚至可在完全没准备好操作的情况下
唤醒waker。除了浪费些CPU周期外并没有其他错误，这种实现方式会导致更频繁循环过程。

## 更新 Mini Tokio

下一步是更新 Mini Tokio 以接收 waker 的通知。我们想让执行器只执行那些唤醒的任务，为此，
Mini Tokio 将提供自己的waker。当waker被调用，与其关联的任务将被加入运行时执行队列。
Mini-Tokio 会在poll这个future时将waker传给它。

更新的 Mini Tokio 使用管道来存储调度的任务。管道允许被缓存的任务能被切换到任何线程执行。
Wakers 需要能被 `Send` 和 `Sync`，所以我们使用crossbeam crate的管道，因为标准库的管道
是支持 `Sync`的。

[[提示]]
| `Send` 和 `Sync` traits 是与Rust并发相关的标记 traits。能被 **sent** 不同线程的
| 类型叫作 `Send`。大多数类型是可 `Send`的，但有些如 [`Rc`] 就不可以。那些能被用不可变
| 引用 **并发**访问的类型就是可 `Sync`的。有的类型能 `Send` 但不能 `Sync` 
| ———— 例如 [`Cell`]， 因它能被不可变引用修改，并发访问它将是不安全的。 
|
| 更多细节，可看[Rust book][ch]的相关章节。

[`Rc`]: https://doc.rust-lang.org/std/rc/struct.Rc.html
[`Cell`]: https://doc.rust-lang.org/std/cell/struct.Cell.html
[ch]: https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html

将如下依赖添加到你的 `Cargo.toml` 以获得管道。

```toml
crossbeam = "0.8"
```

然后更新 `MiniTokio` struct.

```rust
use crossbeam::channel;
use std::sync::Arc;

struct MiniTokio {
    scheduled: channel::Receiver<Arc<Task>>,
    sender: channel::Sender<Arc<Task>>,
}

struct Task {
    // This will be filled in soon.
}
```

Wakers 是 `Sync` 的也能被克隆。当调用 `wake` 时，任务需要被调度并执行。为此，
我们需要一个管道。当调用waker的 `wake()`时，任务被送进管道的发送端。我们的
`Task` 类型将实现wake逻辑。为此，它需要同时包含生成的future和管道发送端。

```rust
# use std::future::Future;
# use std::pin::Pin;
# use crossbeam::channel;
use std::sync::{Arc, Mutex};

struct Task {
    // The `Mutex` is to make `Task` implement `Sync`. Only
    // one thread accesses `future` at any given time. The
    // `Mutex` is not required for correctness. Real Tokio
    // does not use a mutex here, but real Tokio has
    // more lines of code than can fit in a single tutorial
    // page.
    future: Mutex<Pin<Box<dyn Future<Output = ()> + Send>>>,
    executor: channel::Sender<Arc<Task>>,
}

impl Task {
    fn schedule(self: &Arc<Self>) {
        self.executor.send(self.clone());
    }
}
```

为调度这个任务，`Arc` 任务被克隆然后通过管道发送。现在，我们需要用[`std::task::Waker`][`Waker`]
来修改 `schedule` 函数。标准库提供了一个low-level API来用[manual vtable construction][vtable]实现这个。
这为实现者提供了最大的弹性，但需要用一堆不安全的样例代码。所以不必直接使用[`RawWakerVTable`][vtable]，我们
可使用[`futures`] crate提供的[`ArcWake`] 工具类。这让我们可用一个简单 trait 实现以将我们
`Task` 结构体也作为一个waker。

添加如下依赖到你的 `Cargo.toml` 以获得 `futures`。

```toml
futures = "0.3"
```

然后实现 [`futures::task::ArcWake`][`ArcWake`].

```rust
use futures::task::{self, ArcWake};
use std::sync::Arc;
# struct Task {}
# impl Task {
#     fn schedule(self: &Arc<Self>) {}
# }
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.schedule();
    }
}
```

当从上面的timer线程调用 `waker.wake()`时，这个任务将被发送到管道。接着，
我们实现接收和执行任务的函数`MiniTokio::run()`。

```rust
# use crossbeam::channel;
# use futures::task::{self, ArcWake};
# use std::future::Future;
# use std::pin::Pin;
# use std::sync::{Arc, Mutex};
# use std::task::{Context};
# struct MiniTokio {
#   scheduled: channel::Receiver<Arc<Task>>,
#   sender: channel::Sender<Arc<Task>>,
# }
# struct Task {
#   future: Mutex<Pin<Box<dyn Future<Output = ()> + Send>>>,
#   executor: channel::Sender<Arc<Task>>,
# }
# impl ArcWake for Task {
#   fn wake_by_ref(arc_self: &Arc<Self>) {}
# }
impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    /// Initialize a new mini-tokio instance.
    fn new() -> MiniTokio {
        let (sender, scheduled) = channel::unbounded();

        MiniTokio { scheduled, sender }
    }

    /// Spawn a future onto the mini-tokio instance.
    ///
    /// The given future is wrapped with the `Task` harness and pushed into the
    /// `scheduled` queue. The future will be executed when `run` is called.
    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        Task::spawn(future, &self.sender);
    }
}

impl Task {
    fn poll(self: Arc<Self>) {
        // Create a waker from the `Task` instance. This
        // uses the `ArcWake` impl from above.
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);

        // No other thread ever tries to lock the future
        let mut future = self.future.try_lock().unwrap();

        // Poll the future
        let _ = future.as_mut().poll(&mut cx);
    }

    // Spawns a new task with the given future.
    //
    // Initializes a new Task harness containing the given future and pushes it
    // onto `sender`. The receiver half of the channel will get the task and
    // execute it.
    fn spawn<F>(future: F, sender: &channel::Sender<Arc<Task>>)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            future: Mutex::new(Box::pin(future)),
            executor: sender.clone(),
        });

        let _ = sender.send(task);
    }

}
```

这里发生了好多事。首先，实现了`MiniTokio::run()`。这个函数运行了一个从管道中接收调度任务的循环。
所有的任务在唤醒后被发送到管道中，这些任务在执行后能够有状态迁移。

此外，函数 `MiniTokio::new()` 和 `MiniTokio::spawn()` 被调整为改用管道而不是 `VecDeque`。
当新任务被生成后，它们被发送到管道发送端的克隆，任务后续可以用这个发送端来继续将自己调度到运行时执行。

函数`Task::poll()` 使用`futures` crate 的 [`ArcWake`] 工具类来创建waker。这个waker被用来生成
`task::Context`，进而传给future的 `poll`函数。

# 总结

我们现在看到了完整的Rust异步编程样例代码。Rust的`async/await` 功能是通过traits来支撑的。
这允许第三方crates，如Tokio来提供执行过程细节。

* Rust 异步操作是lazy的，需要一个调用方通过poll函数来调用它们。
* Wakers被传给futures，和关联的future一起对应到任务上。
* 当资源 **没有** 就绪时，直到可以操作完成前，返回 `Poll::Pending` 并记下唤醒任务的waker。
* 当资源就绪，任务的waker会收到唤醒
* 执行器在收到通知后，调度任务去执行
* 任务会再次被poll，这次资源是就绪的，所以任务执行后将迁移到新状态。

# 一些延伸

回忆我们在实现 `Delay` future时，我们说有些缺陷问题需要修复。Rust异步模型运行单个future
执行时会在任务间迁移。考虑如下：

```rust
use futures::future::poll_fn;
use std::future::Future;
use std::pin::Pin;
# use std::task::{Context, Poll};
# use std::time::{Duration, Instant};
# struct Delay { when: Instant }
# impl Future for Delay {
#   type Output = ();
#   fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<()> {
#       Poll::Pending
#   }  
# }

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let mut delay = Some(Delay { when });

    poll_fn(move |cx| {
        let mut delay = delay.take().unwrap();
        let res = Pin::new(&mut delay).poll(cx);
        assert!(res.is_pending());
        tokio::spawn(async move {
            delay.await;
        });

        Poll::Ready(())
    }).await;
}
```

函数 `poll_fn` 用闭包创建一个 `Future` 实例。上面这段代码创建一个`Delay` 实例，
并用poll调用它一次，然后将 `Delay` 实例发送到一个新任务中await。在这个例子中，多次
利用 **不同的** `Waker` 实例调用`Delay::poll`。当这样使用了，你需要确保在`Waker`
上调用 `wake` 传递到_最近的_`poll`调用。

当实现future时，假设每次调用`poll`**可以**供应一个不同的`Waker` 实例是很关键的。函数
poll要将之前记下的任务waker更换成一个新的。

我们之前的`Delay` 实现每次poll时生成一个新线程。这可以，但频繁调用poll就不是很有效率
(e.g. 如果你 `select!` 遍历一组futures，如果任何一个future有新事件发送其他futures都会被poll到)。
一个方法是，记住是否生成过新线程，如果没有生成过，就生成一个新的。尽管可以这么做，你也
需要确保线程的`Waker`在后面的poll调用时是更新的，否则就没有唤醒到最近的`Waker`。

为修复之前的实现，我们可以这么做:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
    // This is Some when we have spawned a thread, and None otherwise.
    waker: Option<Arc<Mutex<Waker>>>,
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // First, if this is the first time the future is called, spawn the
        // timer thread. If the timer thread is already running, ensure the
        // stored `Waker` matches the current task's waker.
        if let Some(waker) = &self.waker {
            let mut waker = waker.lock().unwrap();

            // Check if the stored waker matches the current task's waker.
            // This is necessary as the `Delay` future instance may move to
            // a different task between calls to `poll`. If this happens, the
            // waker contained by the given `Context` will differ and we
            // must update our stored waker to reflect this change.
            if !waker.will_wake(cx.waker()) {
                *waker = cx.waker().clone();
            }
        } else {
            let when = self.when;
            let waker = Arc::new(Mutex::new(cx.waker().clone()));
            self.waker = Some(waker.clone());

            // This is the first time `poll` is called, spawn the timer thread.
            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                // The duration has elapsed. Notify the caller by invoking
                // the waker.
                let waker = waker.lock().unwrap();
                waker.wake_by_ref();
            });
        }

        // Once the waker is stored and the timer thread is started, it is
        // time to check if the delay has completed. This is done by
        // checking the current instant. If the duration has elapsed, then
        // the future has completed and `Poll::Ready` is returned.
        if Instant::now() >= self.when {
            Poll::Ready(())
        } else {
            // The duration has not elapsed, the future has not completed so
            // return `Poll::Pending`.
            //
            // The `Future` trait contract requires that when `Pending` is
            // returned, the future ensures that the given waker is signalled
            // once the future should be polled again. In our case, by
            // returning `Pending` here, we are promising that we will
            // invoke the given waker included in the `Context` argument
            // once the requested duration has elapsed. We ensure this by
            // spawning the timer thread above.
            //
            // If we forget to invoke the waker, the task will hang
            // indefinitely.
            Poll::Pending
        }
    }
}
```

稍微有点绕，主要意思是，每次调用到 `poll`, 这个future检查对应的waker是否是前面记下的waker，
如果是同一个，就不需要做别的。如果不是，就更换它（future对应的waker）。

## `Notify` 工具

我们展示了如何采用 waker 来实现`Delay` future。Wakers 是异步Rust工作的核心。通常，不需要
深入到这个层面。例如，在`Delay`这个例子上，我们可以采用[`tokio::sync::Notify`][notify] 
工具类来实现它的`async/await`能力。这个工具类提供基本的任务通知机制。它处理waker的细节，
包括确认future对应的waker与当前task的匹配。

通过 [`Notify`][notify]，我们可以用`async/await`实现一个 `delay` 函数如下:

```rust
use tokio::sync::Notify;
use std::sync::Arc;
use std::time::{Duration, Instant};
use std::thread;

async fn delay(dur: Duration) {
    let when = Instant::now() + dur;
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    thread::spawn(move || {
        let now = Instant::now();

        if now < when {
            thread::sleep(when - now);
        }

        notify2.notify_one();
    });


    notify.notified().await;
}
```

[assoc]: https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types
[trait]: https://doc.rust-lang.org/std/future/trait.Future.html
[pin]: https://doc.rust-lang.org/std/pin/index.html
[`Waker`]: https://doc.rust-lang.org/std/task/struct.Waker.html
[mini-tokio]: https://github.com/tokio-rs/website/blob/master/tutorial-code/mini-tokio/src/main.rs
[vtable]: https://doc.rust-lang.org/std/task/struct.RawWakerVTable.html
[`ArcWake`]: https://docs.rs/futures/0.3/futures/task/trait.ArcWake.html
[`futures`]: https://docs.rs/futures/
[notify]: https://docs.rs/tokio/1/tokio/sync/struct.Notify.html

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
它由标准库提供。它包含了待处理的异步计算的值。

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

[关联类型][assoc] `Output` 值是由future在完成后产生的。[`Pin`][pin] 类型是Rust如何
支持在`async` 函数中借用变量的基础。从 [标准库][pin] 文档中参考更多细节。

不像其他语言中的futures实现，Rust的future并不表示一个在背后运行的计算过程，相反Rust的future
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
            // Ignore this line for now.
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

在main函数中，我们实例化了future 并调用 `.await`。在异步函数里，我们可以在任何实现过`Future`
的值上调用 `.await`。相应的，调用`async` 函数返回实现了`Future`的匿名类型值。而调用 `async fn main()`，
大约生成如下future:

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
这future 从状态 `State0` 开始。当调用其 `poll` 方法时，future尝试尽可能迁移其内部状态。如果feature
能执行完，就将这异步执行的结果封装在状态 `Poll::Ready` 里一起返回。

如果该future还无法完成，通常因为还在等待某些未就绪的资源，那么返回 `Poll::Pending` 。当收到
`Poll::Pending` 返回值时，表示该future将会在以后某个时间点完成，调用者需要在后面继续调用 `poll`。

我们也需要注意有些futures会组合其他futures。调用外层future的`poll`会导致进一步调用到内层future的相应函数。

# Executors 执行器

异步的Rust函数返回futures。Futures 必须被调用 `poll` 函数来转换他们的状态。Futures可以由其他futures组合而成。
所以问题是由什么在调用最外层future的 `poll` 函数？

回忆前面，为执行异步函数，它们需要被传递给 `tokio::spawn` 或者对main函数注释为`#[tokio::main]`。
这导致将外层future提交给Tokio执行器。执行器负责调用外层 `Future::poll` 驱动异步函数过程执行到完成状态。

## Mini Tokio

为了更好理解这些完整过程，我们先实现一个最小化的Tokio。完整代码在 [这里][mini-tokio].

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

这里执行了异步代码。一个 `Delay` 实例随指定延迟被生成并await调用。尽管这样，截至目前我们
的实现有一个主要**缺陷**。执行器从不进入休眠。执行器持续遍历全部被生成的futures并poll调用它们。
大多数时候，futures 并不会是就绪状态等着进一步执行，而是会一再返回 `Poll::Pending` 。这个进程
会消耗CPU周期，并不是太有效率。

理想情况，我们希望 mini-tokio 只在 future 能被改变状态时去 poll 它。这会在对应阻塞等待的资源就绪
能进一步操作时才出现。如果该任务实现从TCP socket读取数据，我们只会在TCP socket上收到数据时poll这个任务。
在我们的例子中，这个任务阻塞在指定的 `Instant` 到达时。相应的，mini-tokio 只会在时刻到时 poll该任务。

为实现这个过程，当一个资源被poll了但其状态**还没有** 就绪，这资源会在它的状态迁移到就绪时立刻发出一个通知。

# Wakers 唤醒者

缺的就是 Wakers。这就是资源能够唤醒等待资源就绪时能够通知等待中的任务的机制。 

让我们再看看 `Future::poll` 的定义:

```rust,compile_fail
fn poll(self: Pin<&mut Self>, cx: &mut Context)
    -> Poll<Self::Output>;
```

`Context` 的`poll` 参数有 `waker()` 方法。这个方法会返回一个[`Waker`] 约束在当前任务上. [`Waker`] 有一个 `wake()` 方法。
调用这个方法，会传递信号告诉执行器与它关联的任务应该被再次调度执行了。资源调用 `wake()` 当它们转换到就绪状态以通知执行器此时
来poll这个任务可以转换到新状态。

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

现在，一旦指定的时间过去，调用中的任务被通知，执行器会确保该任务再次调度执行。下一步是
更新 mini-tokio 去监听唤醒通知。

我们 `Delay` 实现仍有一些遗留问题，将在后面再修复它们。

[[警告]]
| 当future 返回 `Poll::Pending`，它**必须**确定waker会再某个时间点被唤醒。 
| 忘记这个步骤会导致任务被无限期挂起。
|
| 在返回 `Poll::Pending` 后忘记唤醒任务是常见错误。

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
因为返回 `Poll::Pending`，我们有责任给waker信号。因为没有额外的timer线程，我们直接在本线程
内部唤醒waker。这么做会导致future被立刻重调度执行。很可能还是没有准备好。

注意，你可以被允许比实际需要更频繁的唤醒waker。在有些情况下，我们甚至在完全没准备好操作的情况下唤醒waker。
除了浪费些CPU周期外并没有其他错误。这样的实现会导致更繁忙的循环。

## 更新 Mini Tokio

下一步是更新 Mini Tokio 以接收 waker 的通知。我们想让执行器只执行那些唤醒的任务，为此，
Mini Tokio 将提供自己的waker。当waker被调用，与其关联的任务将被加入执行队列。
Mini-Tokio 在poll这个future时将waker传给它。

更新的 Mini Tokio 使用管道来存储调度的任务。管道允许被缓存的任务能被任何线程执行。
Wakers 需要能被 `Send` 和 `Sync`，所以我们使用crossbeam crate的管道，因为标准库的
的管道是不可 `Sync`的。

[[提示]]
| `Send` 和 `Sync` traits 与Rust并发相关的标记 traits。能被 **sent** 不同线程的
| 类型叫作 `Send`。大多数类型是可 `Send`的，但有些比如 [`Rc`] 就不可以。那些能被
| 通过不可变引用 **并发**访问的类型就是可 `Sync`的。有的类型能 `Send` 但不能 `Sync` 
| —— 例如 [`Cell`]， 它能被不可变引用修改，并发访问将是不安全的。 
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

Wakers 是 `Sync` 也能被克隆。当调用 `wake` 时，任务需要被调度并执行。为此，
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

为调度这个任务，`Arc` 被克隆然后通过管道发送。现在，我们需要用[`std::task::Waker`][`Waker`]
修改 `schedule` 函数。标准库提供了一个low-level API来用[manual vtable construction][vtable]实现这个。
这为实现者提供了最大的弹性，但需要一堆不安全的样例代码。不直接使用[`RawWakerVTable`][vtable]，我们
将使用[`futures`] crate提供的[`ArcWake`] 工具类。这让我们实现一个简单的trait以将我们
`Task` 结构体暴露为一个waker。

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

当上面的timer线程调用 `waker.wake()`时，这个任务将被发送到管道。接着，
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

Multiple things are happening here. First, `MiniTokio::run()` is implemented.
The function runs in a loop receiving scheduled tasks from the channel.
As tasks are pushed into the channel when they are woken, these tasks are able
to make progress when executed.

Additionally, the `MiniTokio::new()` and `MiniTokio::spawn()` functions are
adjusted to use a channel rather than a `VecDeque`. When new tasks are spawned,
they are given a clone of the sender-part of the channel, which the task can
use to schedule itself on the runtime.

The `Task::poll()` function creates the waker using the [`ArcWake`] utility from
the `futures` crate. The waker is used to create a `task::Context`. That
`task::Context` is passed to `poll`.

# Summary

We have now seen an end-to-end example of how asynchronous Rust works. Rust's
`async/await` feature is backed by traits. This allows third-party crates, like
Tokio, to provide the execution details.

* Asynchronous Rust operations are lazy and require a caller to poll them.
* Wakers are passed to futures to link a future to the task calling it.
* When a resource is **not** ready to complete an operation, `Poll::Pending` is
  returned and the task's waker is recorded.
* When the resource becomes ready, the task's waker is notified.
* The executor receives the notification and schedules the task to execute.
* The task is polled again, this time the resource is ready and the task makes
  progress.

# A few loose ends

Recall when we were implementing the `Delay` future, we said there were a few
more things to fix. Rust's asynchronous model allows a single future to migrate
across tasks while it executes. Consider the following:

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

The `poll_fn` function creates a `Future` instance using a closure. The snippet
above creates a `Delay` instance, polls it once, then sends the `Delay` instance
to a new task where it is awaited. In this example, `Delay::poll` is called more
than once with **different** `Waker` instances. When this happens, you must make
sure to call `wake` on the `Waker` passed to _the most recent_ call to `poll`.

When implementing a future, it is critical to assume that each call to `poll`
**could** supply a different `Waker` instance. The poll function must update any
previously recorded waker with the new one.

Our earlier implementation of `Delay` spawned a new thread every time it was
polled. This is fine, but can be very inefficient if it is polled too often
(e.g. if you `select!` over that future and some other future, both are polled
whenever either has an event). One approach to this is to remember whether you
have already spawned a thread, and only spawn a new thread if you haven't
already spawned one.  However if you do this, you must ensure that the thread's
`Waker` is updated on later calls to poll, as you are otherwise not waking the
most recent `Waker`.

To fix our earlier implementation, we could do something like this:

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

It is a bit involved, but the idea is, on each call to `poll`, the future checks
if the supplied waker matches the previously recorded waker. If the two wakers
match, then there is nothing else to do. If they do not match, then the recorded
waker must be updated.

## `Notify` utility

We demonstrated how a `Delay` future could be implemented by hand using wakers.
Wakers are the foundation of how asynchronous Rust works. Usually, it is not
necessary to drop down to that level. For example, in the case of `Delay`, we
could implement it entirely with `async/await` by using the
[`tokio::sync::Notify`][notify] utility. This utility provides a basic task
notification mechanism. It handles the details of wakers, including making sure
that the recorded waker matches the current task.

Using [`Notify`][notify], we can implement a `delay` function using
`async/await` like this:

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

---
title: "共享数据状态"
---

截至目前我们实现了一个key-value服务器。但它有一个主要缺陷: 数据状态不能在不同的连接之间共享。
本节我们将修复这个问题。

# Strategies 策略

Tokio中有许多不同的方案来共享状态。

1. 采用互斥锁来管理共享状态。
2. 生成任务来管理，采用管道消息来驱动。

通常你可以用第1种方法来保护数据结构，用第2种方法来处理I/O原语的异步操作。本节要保护的
状态是`HashMap` 和其 `insert`、`get`操作，这些都不是异步操作，我们用`Mutex`保护它。

第2种方法我们将在下一节探讨。

# 添加 `bytes` 库 

Mini-Redis crate 使用 [`bytes`]库里的`Bytes`代替标准库的`Vec<u8>`。`Bytes` 能为网络编程
提供健壮的字节数组结构。它在 `Vec<u8>` 之上添加的最大特性是浅拷贝。换句话说调用 `clone()` 
不会拷贝底层数据。`Bytes` 实例是底层数据的引用计数句柄。`Bytes` 类型除了其他额外功能外，
粗略等同于 `Arc<Vec<u8>>` 。

为使用 `bytes`, 添加如下内容到 `Cargo.toml` 的 `[dependencies]` 部分:

```toml
bytes = "1"
```

[`bytes`]: https://docs.rs/bytes/1/bytes/struct.Bytes.html

# 初始化 `HashMap`

`HashMap` 将在多个任务中共享，可能是在多个线程中运行。为支持它需要用到 `Arc<Mutex<_>>`。

首先，为了方便，在 `use` 语句后定义对应类型。

```rust
use bytes::Bytes;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

type Db = Arc<Mutex<HashMap<String, Bytes>>>;
```

然后，更新 `main` 函数，初始化`HashMap` 并传 `Arc` **handle** 给 `process` 函数。
使用`Arc` 让 `HashMap`成为可在多并发任务及线程里使用的引用。在Tokio中，术语 **handle** 
被用于引用那些共享状态值。

```rust
use tokio::net::TcpListener;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

# fn dox() {
#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    println!("Listening");

    let db = Arc::new(Mutex::new(HashMap::new()));

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // Clone the handle to the hash map.
        let db = db.clone();

        println!("Accepted");
        tokio::spawn(async move {
            process(socket, db).await;
        });
    }
}
# }
# type Db = Arc<Mutex<HashMap<(), ()>>>;
# async fn process(_: tokio::net::TcpStream, _: Db) {}
```

## 使用 `std::sync::Mutex`

注意，是用 `std::sync::Mutex` 而非 **not** `tokio::sync::Mutex` 来保护`HashMap`。
常见的错误是，在异步代码中无条件的用 `tokio::sync::Mutex`。异步锁适合用在跨`.await`调用的环境。

同步锁会阻塞当前任务线程，相应的就阻塞了其他任务的执行。尽管如此，切换到`tokio::sync::Mutex`
通常也并不可行，因为异步锁的内部也是用同步锁实现。

一个经验法则是，只要锁竞争小且不会跨`.await`调用的话，就可以在异步代码中使用同步锁，
也可以用更快的 [`parking_lot::Mutex`][parking_lot] 作为 `std::sync::Mutex`的替代。

[parking_lot]: https://docs.rs/parking_lot/0.10.2/parking_lot/type.Mutex.html

# 更新 `process()`

在函数process 不在初始化 `HashMap`，它使用`HashMap`的共享句柄作为参数。在使用
`HashMap` 前需要加锁。注意HashMap的值类型是 `Bytes`了 (轻量克隆)，相应修改。

```rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};
# use std::collections::HashMap;
# use std::sync::{Arc, Mutex};
# type Db = Arc<Mutex<HashMap<String, bytes::Bytes>>>;

async fn process(socket: TcpStream, db: Db) {
    use mini_redis::Command::{self, Get, Set};

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                let mut db = db.lock().unwrap();
                db.insert(cmd.key().to_string(), cmd.value().clone());
                Frame::Simple("OK".to_string())
            }           
            Get(cmd) => {
                let db = db.lock().unwrap();
                if let Some(value) = db.get(cmd.key()) {
                    Frame::Bulk(value.clone())
                } else {
                    Frame::Null
                }
            }
            cmd => panic!("unimplemented {:?}", cmd),
        };

        // Write the response to the client
        connection.write_frame(&response).await.unwrap();
    }
}
```

# Tasks, threads, and contention 任务，线程，锁竞争

Using a blocking mutex to guard short critical sections is an acceptable
strategy when contention is minimal. When a lock is contended, the thread
executing the task must block and wait on the mutex. This will not only block
the current task but it will also block all other tasks scheduled on the current
thread.

By default, the Tokio runtime uses a multi-threaded scheduler. Tasks are
scheduled on any number of threads managed by the runtime. If a large number of
tasks are scheduled to execute and they all require access to the mutex, then
there will be contention. On the other hand, if the
[`current_thread`][current_thread] runtime flavor is used, then the mutex will
never be contended.

[[info]]
| The [`current_thread` runtime flavor][basic-rt] is a lightweight,
| single-threaded runtime. It is a good choice when only spawning
| a few tasks and opening a handful of sockets. For example, this
| option works well when providing a synchronous API bridge on top
| of an asynchronous client library.

[basic-rt]: https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_current_thread

If contention on a synchronous mutex becomes a problem, the best fix is rarely
to switch to the Tokio mutex. Instead, options to consider are:

- Switching to a dedicated task to manage state and use message passing.
- Shard the mutex.
- Restructure the code to avoid the mutex.

In our case, as each *key* is independent, mutex sharding will work well. To do
this, instead of having a single `Mutex<HashMap<_, _>>` instance, we would
introduce `N` distinct instances.

```rust
# use std::collections::HashMap;
# use std::sync::{Arc, Mutex};
type ShardedDb = Arc<Vec<Mutex<HashMap<String, Vec<u8>>>>>;

fn new_sharded_db(num_shards: usize) -> ShardedDb {
    let mut db = Vec::with_capacity(num_shards);
    for _ in 0..num_shards {
        db.push(Mutex::new(HashMap::new()));
    }
    Arc::new(db)
}
```

Then, finding the cell for any given key becomes a two step process. First, the
key is used to identify which shard it is part of. Then, the key is looked up in
the `HashMap`.

```rust,compile_fail
let shard = db[hash(key) % db.len()].lock().unwrap();
shard.insert(key, value);
```

The simple implementation outlined above requires using a fixed number of
shards, and the number of shards cannot be changed once the sharded map is
created. The [dashmap] crate provides an implementation of a more sophisticated
sharded hash map.

[current_thread]: https://docs.rs/tokio/1/tokio/runtime/index.html#current-thread-scheduler
[dashmap]: https://docs.rs/dashmap

# Holding a `MutexGuard` across an `.await`

You might write code that looks like this:
```rust
use std::sync::{Mutex, MutexGuard};

async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
# async fn do_something_async() {}
```
When you try to spawn something that calls this function, you will encounter the
following error message:
```text
error: future cannot be sent between threads safely
   --> src/lib.rs:13:5
    |
13  |     tokio::spawn(async move {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-0.2.21/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/lib.rs:7:5
    |
4   |     let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    |         -------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
7   |     do_something_async().await;
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here, with `mut lock` maybe used later
8   | }
    | - `mut lock` is later dropped here
```
This happens because the `std::sync::MutexGuard` type is **not** `Send`. This
means that you can't send a mutex lock to another thread, and the error happens
because the Tokio runtime can move a task between threads at every `.await`.
To avoid this, you should restructure your code such that the mutex lock's
destructor runs before the `.await`.
```rust
# use std::sync::{Mutex, MutexGuard};
// This works!
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    {
        let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
        *lock += 1;
    } // lock goes out of scope here

    do_something_async().await;
}
# async fn do_something_async() {}
```
Note that this does not work:
```rust
use std::sync::{Mutex, MutexGuard};

// This fails too.
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;
    drop(lock);

    do_something_async().await;
}
# async fn do_something_async() {}
```
This is because the compiler currently calculates whether a future is `Send`
based on scope information only. The compiler will hopefully be updated to
support explicitly dropping it in the future, but for now, you must explicitly
use a scope.

Note that the error discussed here is also discussed in the [Send bound section
from the spawning chapter][send-bound].

You should not try to circumvent this issue by spawning the task in a way that
does not require it to be `Send`, because if Tokio suspends your task at an
`.await` while the task is holding the lock, some other task may be scheduled to
run on the same thread, and this other task may also try to lock that mutex,
which would result in a deadlock as the task waiting to lock the mutex would
prevent the task holding the mutex from releasing the mutex.

We will discuss some approaches to fix the error message below:

[send-bound]: spawning#send-bound

## Restructure your code to not hold the lock across an `.await`

We have already seen one example of this in the snippet above, but there are
some more robust ways to do this. For example, you can wrap the mutex in a
struct, and only ever lock the mutex inside non-async methods on that struct.
```rust
use std::sync::Mutex;

struct CanIncrement {
    mutex: Mutex<i32>,
}
impl CanIncrement {
    // This function is not marked async.
    fn increment(&self) {
        let mut lock = self.mutex.lock().unwrap();
        *lock += 1;
    }
}

async fn increment_and_do_stuff(can_incr: &CanIncrement) {
    can_incr.increment();
    do_something_async().await;
}
# async fn do_something_async() {}
```
This pattern guarantees that you won't run into the `Send` error, because the
mutex guard does not appear anywhere in an async function.

## Spawn a task to manage the state and use message passing to operate on it

This is the second approach mentioned in the start of this chapter, and is often
used when the shared resource is an I/O resource. See the next chapter for more
details.

## Use Tokio's asynchronous mutex

The [`tokio::sync::Mutex`] type provided by Tokio can also be used. The primary
feature of the Tokio mutex is that it can be held across an `.await` without any
issues. That said, an asynchronous mutex is more expensive than an ordinary
mutex, and it is typically better to use one of the two other approaches.
```rust
use tokio::sync::Mutex; // note! This uses the Tokio mutex

// This compiles!
// (but restructuring the code would be better in this case)
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock = mutex.lock().await;
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
# async fn do_something_async() {}
```

[`tokio::sync::Mutex`]: https://docs.rs/tokio/1/tokio/sync/struct.Mutex.html

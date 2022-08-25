---
title: "Spawning"
---

我们将继续编写Redis服务器。

首先将客户端 `SET`/`GET` 代码迁移到examples目录下，这样可以单独执行它。

```bash
$ mkdir -p examples
$ mv src/main.rs examples/hello-redis.rs
```

接着创建一个新的 `src/main.rs` 空文件。

# 接收socket连接

我们的Redis服务器需要做的第一件事就是接收进入的TCP sockets。可使用 [`tokio::net::TcpListener`][tcpl]。

[[info]]
| 很多Tokio类型命名与Rust标准库中的版本相同。这样Tokio暴露了与标准库`std` 相同的API但却是`async fn`异步函数。

`TcpListener`绑定在端口**6379**, 在循环中接收socket连接。处理每个socket然后关闭它。我们将继续读command，
并打印到stdout，然后返回error给客户的。

```rust
use tokio::net::{TcpListener, TcpStream};
use mini_redis::{Connection, Frame};

# fn dox() {
#[tokio::main]
async fn main() {
    // Bind the listener to the address
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // The second item contains the IP and port of the new connection.
        let (socket, _) = listener.accept().await.unwrap();
        process(socket).await;
    }
}
# }

async fn process(socket: TcpStream) {
    // The `Connection` lets us read/write redis **frames** instead of
    // byte streams. The `Connection` type is defined by mini-redis.
    let mut connection = Connection::new(socket);

    if let Some(frame) = connection.read_frame().await.unwrap() {
        println!("GOT: {:?}", frame);

        // Respond with an error
        let response = Frame::Error("unimplemented".to_string());
        connection.write_frame(&response).await.unwrap();
    }
}
```

运行服务:

```bash
$ cargo run
```

在另一个终端窗口运行 `hello-redis` 例子 (即前一节实现的 `SET`/`GET` command):

```bash
$ cargo run --example hello-redis
```

输出如下:

```text
Error: "unimplemented"
```

服务端窗口输出如下:

```text
GOT: Array([Bulk(b"set"), Bulk(b"hello"), Bulk(b"world")])
```

[tcpl]: https://docs.rs/tokio/1/tokio/net/struct.TcpListener.html

# Concurrency 并发

这里实现的服务器有个小问题 (除了只响应errors外)，它每次只处理一个进来的请求。每接收一个新请求，
服务器会停留在接收循环中，阻塞到响应结果全写入socket为止。

如果希望Redis server 能处理 **many** 并发请求，我们需要加上一些并发处理能力。

[[info]]
| Concurrency并发 与 parallelism并行 并不是同一回事。如果你能在两个任务之间来回切换，
| 那么你就是在并发的处理两个任务，但不是并行的处理。如果想要完全并行处理，你需要有两个人，每人处理一个。
|
| 使用Tokio的优势之一是，异步代码允许你并发执行很多任务，而不需要像传统方式一样采用很多线程。
| 事实上，Tokio也能在单个线程上并发执行很多任务。

为了并发处理很多连接，每接收一个新连接就生成一个新任务，在任务中处理连接的其他工作。

接收循环（accept loop）如下:

```rust
use tokio::net::TcpListener;

# fn dox() {
#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // A new task is spawned for each inbound socket. The socket is
        // moved to the new task and processed there.
        tokio::spawn(async move {
            process(socket).await;
        });
    }
}
# }
# async fn process(_: tokio::net::TcpStream) {}
```

## Tasks 任务

Tokio任务是异步的绿色线程。它们在传递`async`代码块到`tokio::spawn`时创建。
`tokio::spawn` 函数返回一个`JoinHandle`, 调用者可使用它与生成的任务交互。
`async`代码块有返回值。调用者可以通过使用`JoinHandle`的`.await` 来获得返回值。

例如:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // Do some async work
        "return value"
    });

    // Do some other work

    let out = handle.await.unwrap();
    println!("GOT {}", out);
}
```

等待`JoinHandle` 返回 `Result`。当任务执行中遇到错误时，`JoinHandle` 也将返回 `Err`。
在任务panics时, 或任务在运行时关闭而被取消时，都会返回错误。

任务是调度器在执行时的单位块。生成任务并提交给Tokio调度器，它将确保任务被执行。生成的任务
可能在生成时相同的线程，或在另外一个运行时线程。在生成后，任务可以在不同线程中迁移。

Tokio任务非常轻量级，在内部它只需要占用64字节内存。应用程序可以自由创建上千个任务不必担心资源，
上百万个则另当别论。

## `'static` 约束

When you spawn a task on the Tokio runtime, its type's lifetime must be `'static`. This
means that the spawned task must not contain any references to data owned
outside the task.

[[info]]
| It is a common misconception that `'static` always means "lives forever",
| but this is not the case. Just because a value is `'static` does not mean
| that you have a memory leak. You can read more in [Common Rust Lifetime
| Misconceptions][common-lifetime].

[common-lifetime]: https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program

For example, the following will not compile:

```rust,compile_fail
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];

    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```

Attempting to compile this results in the following error:

```text
error[E0373]: async block may outlive the current function, but
              it borrows `v`, which is owned by the current function
 --> src/main.rs:7:23
  |
7 |       task::spawn(async {
  |  _______________________^
8 | |         println!("Here's a vec: {:?}", v);
  | |                                        - `v` is borrowed here
9 | |     });
  | |_____^ may outlive borrowed value `v`
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:7:17
  |
7 |       task::spawn(async {
  |  _________________^
8 | |         println!("Here's a vector: {:?}", v);
9 | |     });
  | |_____^
help: to force the async block to take ownership of `v` (and any other
      referenced variables), use the `move` keyword
  |
7 |     task::spawn(async move {
8 |         println!("Here's a vec: {:?}", v);
9 |     });
  |
```

This happens because, by default, variables are not **moved** into async blocks.
The `v` vector remains owned by the `main` function. The `println!` line borrows
`v`. The rust compiler helpfully explains this to us and even suggests the fix!
Changing line 7 to `task::spawn(async move {` will instruct the compiler to
**move** `v` into the spawned task. Now, the task owns all of its data, making
it `'static`.

If a single piece of data must be accessible from more than one task
concurrently, then it must be shared using synchronization primitives such as
`Arc`.

Note that the error message talks about the argument type *outliving* the
`'static` lifetime. This terminology can be rather confusing because the
`'static` lifetime lasts until the end of the program, so if it outlives it,
don't you have a memory leak? The explanation is that it is the *type*, not the
*value* that must outlive the `'static` lifetime, and the value may be destroyed
before its type is no longer valid.

When we say that a value is `'static`, all that means is that it would not be
incorrect to keep that value around forever. This is important because the
compiler is unable to reason about how long a newly spawned task stays around,
so the only way it can be sure that the task doesn't live too long is to make
sure it may live forever.

The article that the info-box earlier links to uses the terminology "bounded by
`'static`" rather than "its type outlives `'static`" or "the value is `'static`"
to refer to `T: 'static`. These all mean the same thing, but are different from
"annotated with `'static`" as in `&'static T`.

## `Send` bound

Tasks spawned by `tokio::spawn` **must** implement `Send`. This allows the Tokio
runtime to move the tasks between threads while they are suspended at an
`.await`.

Tasks are `Send` when **all** data that is held **across** `.await` calls is
`Send`. This is a bit subtle. When `.await` is called, the task yields back to
the scheduler. The next time the task is executed, it resumes from the point it
last yielded. To make this work, all state that is used **after** `.await` must
be saved by the task. If this state is `Send`, i.e. can be moved across threads,
then the task itself can be moved across threads. Conversely, if the state is not
`Send`, then neither is the task.

For example, this works:

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        // The scope forces `rc` to drop before `.await`.
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }

        // `rc` is no longer used. It is **not** persisted when
        // the task yields to the scheduler
        yield_now().await;
    });
}
```

This does not:

```rust,compile_fail
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");

        // `rc` is used after `.await`. It must be persisted to
        // the task's state.
        yield_now().await;

        println!("{}", rc);
    });
}
```

Attempting to compile the snippet results in:

```text
error: future cannot be sent between threads safely
   --> src/main.rs:6:5
    |
6   |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    | 
   ::: [..]spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in
    |                          `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait
    |       `std::marker::Send` is not  implemented for
    |       `std::rc::Rc<&str>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:9
    |
7   |         let rc = Rc::new("hello");
    |             -- has type `std::rc::Rc<&str>` which is not `Send`
...
10  |         yield_now().await;
    |         ^^^^^^^^^^^^^^^^^ await occurs here, with `rc` maybe
    |                           used later
11  |         println!("{}", rc);
12  |     });
    |     - `rc` is later dropped here
```

We will discuss a special case of this error in more depth [in the next
chapter][mutex-guard].

[mutex-guard]: shared-state#holding-a-mutexguard-across-an-await

# Store values

We will now implement the `process` function to handle incoming commands. We
will use a `HashMap` to store values. `SET` commands will insert into the
`HashMap` and `GET` values will load them. Additionally, we will use a loop to
accept more than one command per connection.

```rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};

async fn process(socket: TcpStream) {
    use mini_redis::Command::{self, Get, Set};
    use std::collections::HashMap;

    // A hashmap is used to store data
    let mut db = HashMap::new();

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    // Use `read_frame` to receive a command from the connection.
    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                // The value is stored as `Vec<u8>`
                db.insert(cmd.key().to_string(), cmd.value().to_vec());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                if let Some(value) = db.get(cmd.key()) {
                    // `Frame::Bulk` expects data to be of type `Bytes`. This
                    // type will be covered later in the tutorial. For now,
                    // `&Vec<u8>` is converted to `Bytes` using `into()`.
                    Frame::Bulk(value.clone().into())
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

Now, start the server:

```bash
$ cargo run
```

and in a separate terminal window, run the `hello-redis` example:

```bash
$ cargo run --example hello-redis
```

Now, the output will be:

```text
got value from the server; result=Some(b"world")
```

We can now get and set values, but there is a problem: The values are not
shared between connections. If another socket connects and tries to `GET`
the `hello` key, it will not find anything.

You can find the full code [here][full].

In the next section, we will implement persisting data for all sockets.

[full]: https://github.com/tokio-rs/website/blob/master/tutorial-code/spawning/src/main.rs

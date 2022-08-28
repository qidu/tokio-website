---
title: "I/O"
---

在Tokio中I/O接口基本与系统库`std`中操作相同，但都是不同步的。这里有负责读的 ([`AsyncRead`]) 
特性和负责写的 ([`AsyncWrite`])特性。
指定类型来实现上述特性适合操作 ([`TcpStream`], [`File`],[`Stdout`])。特性[`AsyncRead`] 和 [`AsyncWrite`] 
也被很多数据结构实现了，如 `Vec<u8>` 和 `&[u8]`。这允许在有reader和writer的地方使用字节数组。

本页将继续通过一些例子探讨依靠Tokio的基本I/O读写。下一页深入更高级I/O操作例子。

# `AsyncRead` 和 `AsyncWrite`

这两个特性traits提供了从字节流中异步的读取和写入的能力。那些特性上的方法一般不用直接调用。
与你不会直接调用`Future`的`poll`方法类似。相反，你通过工具类[`AsyncReadExt`] 和 [`AsyncWriteExt`]
使用它们。

我们先简要的看看那些方法。那些函数都是 `async`的，需要结合 `.await`调用。

## `async fn read()`

[`AsyncReadExt::read`][read] 提供了异步读取数据到缓冲中的方法，返回值是读取字节数。

**注意:** 当 `read()` 返回 `Ok(0)`，这表示流已经关闭。继续调用`read()` 都将立即返回 `Ok(0)`。
用在 [`TcpStream`] 实例一起时，表示这个socket的半双工读操作已关闭。

```rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};

# fn dox() {
#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer[..]).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
# }
```

## `async fn read_to_end()`

[`AsyncReadExt::read_to_end`][read_to_end] 从流中读取全部数据直到EOF。

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

# fn dox() {
#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = Vec::new();

    // read the whole file
    f.read_to_end(&mut buffer).await?;
    Ok(())
}
# }
```

## `async fn write()`

[`AsyncWriteExt::write`][write] 将一个数据缓冲写入到writer对象，返回写入的数据量。

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

# fn dox() {
#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    // Writes some prefix of the byte string, but not necessarily all of it.
    let n = file.write(b"some bytes").await?;

    println!("Wrote the first {} bytes of 'some bytes'.", n);
    Ok(())
}
# }
```

## `async fn write_all()`

[`AsyncWriteExt::write_all`][write_all] 将全部缓冲写入到writer对象。

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

# fn dox() {
#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    file.write_all(b"some bytes").await?;
    Ok(())
}
# }
```

这两个traits特性也包含了其他有帮助的方法。可用API文档中得到更综合的列表。

# 帮助函数

此外，就行标准库 `std`， [`tokio::io`] 模块包含很多有帮助的函数就像API一样能与[标准输入][stdin],
[标准输出][stdout] 和 [标准错误][stderr]一起使用。例如，[`tokio::io::copy`][copy] 
能异步地从reader对象到writer对象拷贝所有内容。

```rust
use tokio::fs::File;
use tokio::io;

# fn dox() {
#[tokio::main]
async fn main() -> io::Result<()> {
    let mut reader: &[u8] = b"hello";
    let mut file = File::create("foo.txt").await?;

    io::copy(&mut reader, &mut file).await?;
    Ok(())
}
# }
```

Note that this uses the fact that byte arrays also implement `AsyncRead`.

# Echo server

Let's practice doing some asynchronous I/O. We will be writing an echo server.

The echo server binds a `TcpListener` and accepts inbound connections in a loop.
For each inbound connection, data is read from the socket and written
immediately back to the socket. The client sends data to the server and receives
the exact same data back.

We will implement the echo server twice, using slightly different strategies.

## Using `io::copy()`

To start, we will implement the echo logic using the [`io::copy`][copy] utility.

You can write up this code in a new binary file:

```text
touch src/bin/echo-server-copy.rs
```

That you can launch (or just check the compilation) with:

```text
cargo run --bin echo-server-copy
```

You will be able to try the server using a standard command-line tool such as `telnet`, or by writing
a simple client like the one found in the documentation for [`tokio::net::TcpStream`][tcp_example].

This is a TCP server and needs an accept loop. A new task is spawned to process
each accepted socket.

```rust
use tokio::io;
use tokio::net::TcpListener;

# fn dox() {
#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            // Copy data here
        });
    }
}
# }
```

As seen earlier, this utility function takes a reader and a writer and copies
data from one to the other. However, we only have a single `TcpStream`. This
single value implements **both** `AsyncRead` and `AsyncWrite`. Because
`io::copy` requires `&mut` for both the reader and the writer, the socket cannot
be used for both arguments.

```rust,compile_fail
// This fails to compile
io::copy(&mut socket, &mut socket).await
```

## Splitting a reader + writer

To work around this problem, we must split the socket into a reader handle and a
writer handle. The best way to split a reader/writer combo depends on the
specific type.

Any reader + writer type can be split using the [`io::split`][split] utility.
This function takes a single value and returns separate reader and  writer
handles. These two handles can be used independently, including from separate
tasks.

For example, the echo client could handle concurrent reads and writes like this:

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

# fn dox() {
#[tokio::main]
async fn main() -> io::Result<()> {
    let socket = TcpStream::connect("127.0.0.1:6142").await?;
    let (mut rd, mut wr) = io::split(socket);

    // Write data in the background
    tokio::spawn(async move {
        wr.write_all(b"hello\r\n").await?;
        wr.write_all(b"world\r\n").await?;

        // Sometimes, the rust type inferencer needs
        // a little help
        Ok::<_, io::Error>(())
    });

    let mut buf = vec![0; 128];

    loop {
        let n = rd.read(&mut buf).await?;

        if n == 0 {
            break;
        }

        println!("GOT {:?}", &buf[..n]);
    }

    Ok(())
}
# }
```

Because `io::split` supports **any** value that implements `AsyncRead +
AsyncWrite` and returns independent handles, internally `io::split` uses an
`Arc` and a `Mutex`. This overhead can be avoided with `TcpStream`. `TcpStream`
offers two specialized split functions.

[`TcpStream::split`] takes a **reference** to the stream and returns a reader
and writer handle. Because a reference is used, both handles must stay on the
**same** task that `split()` was called from. This specialized `split` is
zero-cost. There is no `Arc` or `Mutex` needed. `TcpStream` also provides
[`into_split`] which supports handles that can move across tasks at the cost of
only an `Arc`.

Because `io::copy()` is called on the same task that owns the `TcpStream`, we
can use [`TcpStream::split`]. The task that processes the echo logic in the server becomes:

```rust
# use tokio::io;
# use tokio::net::TcpStream;
# fn dox(mut socket: TcpStream) {
tokio::spawn(async move {
    let (mut rd, mut wr) = socket.split();
    
    if io::copy(&mut rd, &mut wr).await.is_err() {
        eprintln!("failed to copy");
    }
});
# }
```

You can find the entire code [here][full_copy].

## Manual copying

Now let's look at how we would write the echo server by copying the data
manually. To do this, we use [`AsyncReadExt::read`][read] and
[`AsyncWriteExt::write_all`][write_all].

The full echo server is as follows:

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

# fn dox() {
#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = vec![0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    // Return value of `Ok(0)` signifies that the remote has
                    // closed
                    Ok(0) => return,
                    Ok(n) => {
                        // Copy the data back to socket
                        if socket.write_all(&buf[..n]).await.is_err() {
                            // Unexpected socket error. There isn't much we can
                            // do here so just stop processing.
                            return;
                        }
                    }
                    Err(_) => {
                        // Unexpected socket error. There isn't much we can do
                        // here so just stop processing.
                        return;
                    }
                }
            }
        });
    }
}
# }
```

(You can put this code into `src/bin/echo-server.rs` and launch it with
`cargo run --bin echo-server`).

Let's break it down. First, since the `AsyncRead` and `AsyncWrite` utilities are
used, the extension traits must be brought into scope.

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
```

## Allocating a buffer

The strategy is to read some data from the socket into a buffer then write the
contents of the buffer back to the socket.

```rust
let mut buf = vec![0; 1024];
```

A stack buffer is explicitly avoided. Recall from [earlier][send], we noted that
all task data that lives across calls to `.await` must be stored by the task. In
this case, `buf` is used across `.await` calls. All task data is stored in a
single allocation. You can think of it as an `enum` where each variant is the
data that needs to be stored for a specific call to `.await`.

If the buffer is represented by a stack array, the internal structure for tasks
spawned per accepted socket might look something like:

```rust,compile_fail
struct Task {
    // internal task fields here
    task: enum {
        AwaitingRead {
            socket: TcpStream,
            buf: [BufferType],
        },
        AwaitingWriteAll {
            socket: TcpStream,
            buf: [BufferType],
        }

    }
}
```

If a stack array is used as the buffer type, it will be stored *inline* in the
task structure. This will make the task structure very big. Additionally, buffer
sizes are often page sized. This will, in turn, make `Task` an awkward size:
`$page-size + a-few-bytes`.

The compiler optimizes the layout of async blocks further than a basic `enum`.
In practice, variables are not moved around between variants as would be
required with an `enum`. However, the task struct size is at least as big as the
largest variable.

Because of this, it is usually more efficient to use a dedicated allocation for
the buffer.

## Handling EOF

When the read half of the TCP stream is shut down, a call to `read()` returns
`Ok(0)`. It is important to exit the read loop at this point. Forgetting to
break from the read loop on EOF is a common source of bugs.

```rust
# use tokio::io::AsyncReadExt;
# use tokio::net::TcpStream;
# async fn dox(mut socket: TcpStream) {
# let mut buf = vec![0_u8; 1024];
loop {
    match socket.read(&mut buf).await {
        // Return value of `Ok(0)` signifies that the remote has
        // closed
        Ok(0) => return,
        // ... other cases handled here
# _ => unreachable!(),
    }
}
# }
```

Forgetting to break from the read loop usually results in a 100% CPU infinite
loop situation. As the socket is closed, `socket.read()` returns immediately.
The loop then repeats forever.

Full code can be found [here][full_manual].

[full_manual]: https://github.com/tokio-rs/website/blob/master/tutorial-code/io/src/echo-server.rs
[full_copy]: https://github.com/tokio-rs/website/blob/master/tutorial-code/io/src/echo-server-copy.rs

[send]: /tokio/tutorial/spawning#send-bound

[`AsyncRead`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html
[`AsyncWrite`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html
[`AsyncReadExt`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html
[`AsyncWriteExt`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html
[`TcpStream`]: https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html
[`File`]: https://docs.rs/tokio/1/tokio/fs/struct.File.html
[`Stdout`]: https://docs.rs/tokio/1/tokio/io/struct.Stdout.html
[read]: https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read
[read_to_end]: https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read_to_end
[write]: https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write
[write_all]: https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_all
[`tokio::io`]: https://docs.rs/tokio/1/tokio/io/index.html
[stdin]: https://docs.rs/tokio/1/tokio/io/fn.stdin.html
[stdout]: https://docs.rs/tokio/1/tokio/io/fn.stdout.html
[stderr]: https://docs.rs/tokio/1/tokio/io/fn.stderr.html
[copy]: https://docs.rs/tokio/1/tokio/io/fn.copy.html
[split]: https://docs.rs/tokio/1/tokio/io/fn.split.html
[`TcpStream::split`]: https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.split
[`into_split`]: https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.into_split
[tcp_example]: https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#examples

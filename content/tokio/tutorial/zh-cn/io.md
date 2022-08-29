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

注意这是因为字节数组也实现了 `AsyncRead`特性的前提。

# Echo 服务器

让我们实现点什么功能来练习异步I/O。我们将编译一个echo服务器。

这个echo服务器会绑定 `TcpListener`，并在一个循环中接收进入的连接。对每个连接，
从它socket上读取数据并立即写入socket上。客户端发送数据并接收到服务器端完整的响应。

我们将采用不用的策略实现echo服务器两次。

## 采用 `io::copy()`

我们将采用 [`io::copy`][copy] 工具来实现echo逻辑。

你可以创建如下文件:

```text
touch src/bin/echo-server-copy.rs
```

用如下命令来运行 (或检查兼容性) :

```text
cargo run --bin echo-server-copy
```

你也可以尝试标准命令行工具如 `telnet`来连接服务器，或用
[`tokio::net::TcpStream`][tcp_example]的例子编写一个简单的客户端。

这是一个有接收连接循环的TCP服务器。每接收一个新连接就生成一个新任务来处理它。

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

如前面，这工具函数可以使用一对reader和writer，并从前者拷贝数据到后者。但我们只有一个
`TcpStream`对象，它同时实现了`AsyncRead` 和 `AsyncWrite`特性。由于`io::copy` 要求
reader和writer都是`&mut`类型参数，这个socket对象就不能用在这里。

```rust,compile_fail
// This fails to compile
io::copy(&mut socket, &mut socket).await
```

## 拆分 reader 和 writer

为解决这个问题，我们将把这个socket拆分成一对reader和writer句柄。拆分成reader/writer的
最佳方式是用特定的组合类型。

任何reader + writer 类型可以用[`io::split`][split] 工具类来拆分。它使用单个参数并返回
一对reader和writer句柄。它们可以被分开在不同的任务中使用。

例如，echo客户端可以采用如下方式并发的使用读和写:

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

由于 `io::split` 支持任何实现了`AsyncRead + AsyncWrite`特性的类型，返回读写独立的句柄，
`io::split`内部使用了`Arc` 和 `Mutex`。 `TcpStream`可以避免这个开销，因为`TcpStream`
提供了两个特殊的split函数。

[`TcpStream::split`] 使用一个stream引用作为参数，并返回一对reader和writer句柄。由于
使用了引用，两个句柄需要用在split它们的相同的任务中。这个特殊的`split`函数调用是零成本的
zero-cost。这里没有`Arc` 或 `Mutex`调用。`TcpStream` 也提供[`into_split`]函数，它返回
的句柄支持以`Arc`成本被迁移到不用的任务中。

因为 `io::copy()` 被在拥有`TcpStream`对象的任务中调用, 我们可以使用 [`TcpStream::split`]。
在服务端处理echo逻辑的任务如下：

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

你可以在[这里][full_copy]得到完整代码。

## 手动拷贝

现在看看如何用手动拷贝数据来实现echo服务器。我们使用 [`AsyncReadExt::read`][read] 和
[`AsyncWriteExt::write_all`][write_all]。

完整的echo服务器代码如下:

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

(你可以将代码放在 `src/bin/echo-server.rs` 并用这个命令启动
`cargo run --bin echo-server`).

让我们来分解过程。首先，由于用到了`AsyncRead` 和 `AsyncWrite`工具类，需要也引入
扩展特性traits。

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
```

## 分配缓冲

这里策略是，从socket中读取数据到缓冲里，然后再将其写出到socket中。

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

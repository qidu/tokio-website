---
title: "组帧"
---

现在我们将应用之前学的I/O操作来实现Mini-Redis的拆组帧层。组帧是将字节流转换成帧结构流。
帧是在两个节点间传输时的数据单位。Redis的帧协议定义如下:

```rust
use bytes::Bytes;

enum Frame {
    Simple(String),
    Error(String),
    Integer(u64),
    Bulk(Bytes),
    Null,
    Array(Vec<Frame>),
}
```

注意这里帧只是由数据组成而没有语义。命令的解析和实现是在更上层进行的。

类比HTTP协议，它的一个帧类似如下：

```rust
# use bytes::Bytes;
# type Method = ();
# type Uri = ();
# type Version = ();
# type HeaderMap = ();
# type StatusCode = ();
enum HttpFrame {
    RequestHead {
        method: Method,
        uri: Uri,
        version: Version,
        headers: HeaderMap,
    },
    ResponseHead {
        status: StatusCode,
        version: Version,
        headers: HeaderMap,
    },
    BodyChunk {
        chunk: Bytes,
    },
}
```

为实现Mini-Redis帧过程，我们将用`Connection` 结构来包装一个`TcpStream`来
读/写 `mini_redis::Frame` 值。

```rust
use tokio::net::TcpStream;
use mini_redis::{Frame, Result};

struct Connection {
    stream: TcpStream,
    // ... other fields here
}

impl Connection {
    /// Read a frame from the connection.
    /// 
    /// Returns `None` if EOF is reached
    pub async fn read_frame(&mut self)
        -> Result<Option<Frame>>
    {
        // implementation here
# unimplemented!();
    }

    /// Write a frame to the connection.
    pub async fn write_frame(&mut self, frame: &Frame)
        -> Result<()>
    {
        // implementation here
# unimplemented!();
    }
}
```

你可以在 [这里][proto]得到Redis详细协议。完整的`Connection` 代码在 [这里][full].

[proto]: https://redis.io/topics/protocol
[full]: https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs

# 有缓冲的读操作

`read_frame` 方法在返回前会等待接收到一个完整的帧。单独调用`TcpStream::read()` 会返回
任意数量的字节。它可能包含一个帧、帧的一部分或几个帧。如果收到的只是帧的一部分，先缓存住
这些数据再继续从socket上读。如果收到了多个帧，先返回第一个帧，并将剩下的帧缓存住等待下一
次调用`read_frame`。

为实现这个功能， `Connection` 需要一个数据缓冲。从socket中读数据到缓冲里。当解析到一个帧，
对应的数据会从缓冲里删除。

我们将使用 [`BytesMut`][BytesMutStruct] 作为缓冲类型。这是可改变数据的[`Bytes`][BytesStruct]
缓冲版本。

```rust
use bytes::BytesMut;
use tokio::net::TcpStream;

pub struct Connection {
    stream: TcpStream,
    buffer: BytesMut,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream,
            // Allocate the buffer with 4kb of capacity.
            buffer: BytesMut::with_capacity(4096),
        }
    }
}
```

接着，实现 `read_frame()` 方法。

```rust
use tokio::io::AsyncReadExt;
use bytes::Buf;
use mini_redis::Result;

# struct Connection {
#   stream: tokio::net::TcpStream,
#   buffer: bytes::BytesMut,
# }
# struct Frame {}
# impl Connection {
pub async fn read_frame(&mut self)
    -> Result<Option<Frame>>
{
    loop {
        // Attempt to parse a frame from the buffered data. If
        // enough data has been buffered, the frame is
        // returned.
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // There is not enough buffered data to read a frame.
        // Attempt to read more data from the socket.
        //
        // On success, the number of bytes is returned. `0`
        // indicates "end of stream".
        if 0 == self.stream.read_buf(&mut self.buffer).await? {
            // The remote closed the connection. For this to be
            // a clean shutdown, there should be no data in the
            // read buffer. If there is, this means that the
            // peer closed the socket while sending a frame.
            if self.buffer.is_empty() {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        }
    }
}
# fn parse_frame(&self) -> Result<Option<Frame>> { unimplemented!() }
# }
```

让我们来分解这个过程。在循环中执行`read_frame`。首先，调用`self.parse_frame()` ，
这会尝试从`self.buffer`中解析一个帧。如果有充足的数据可解析出帧，就将它返回给
`read_frame()`的调用者。否则，我们尝试从socket中读出更多数据保存到缓冲里。如果
读到更多数据，继续调用 `parse_frame()` ，这次要是数据充足，就可以解析出帧。

当从流中读数据时，返回值 `0` 暗示对端不会再有更多数据了。如果缓冲里还有数据，说明
只收到帧的一部分，连接被意外中断了。这满足出错条件，所以返回了`Err`。

[BytesMutStruct]: https://docs.rs/bytes/1/bytes/struct.BytesMut.html
[BytesStruct]: https://docs.rs/bytes/1/bytes/struct.Bytes.html

## `Buf` 特性

当读取数据时，调用了 `read_buf` 。这个函数使用的参数实现了 [`bytes`] crate中的 [`BufMut`] 。

首先，考虑我们如何用`read()`实现同样的读取循环。可用`Vec<u8>` 来代替`BytesMut`。

```rust
use tokio::net::TcpStream;

pub struct Connection {
    stream: TcpStream,
    buffer: Vec<u8>,
    cursor: usize,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream,
            // Allocate the buffer with 4kb of capacity.
            buffer: vec![0; 4096],
            cursor: 0,
        }
    }
}
```

添加 `read_frame()` 函数到`Connection`:

```rust
use mini_redis::{Frame, Result};

# use tokio::io::AsyncReadExt;
# pub struct Connection {
#     stream: tokio::net::TcpStream,
#     buffer: Vec<u8>,
#     cursor: usize,
# }
# impl Connection {
pub async fn read_frame(&mut self)
    -> Result<Option<Frame>>
{
    loop {
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // Ensure the buffer has capacity
        if self.buffer.len() == self.cursor {
            // Grow the buffer
            self.buffer.resize(self.cursor * 2, 0);
        }

        // Read into the buffer, tracking the number
        // of bytes read
        let n = self.stream.read(
            &mut self.buffer[self.cursor..]).await?;

        if 0 == n {
            if self.cursor == 0 {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        } else {
            // Update our cursor
            self.cursor += n;
        }
    }
}
# fn parse_frame(&mut self) -> Result<Option<Frame>> { unimplemented!() }
# }
```

当使用字节数组和 `read`时，我们需要自行维护一个位置变量来跟踪缓冲了多少数据。
我们必须确保 `read()`数据填充到缓冲空间的空白区域。否则，就覆盖已缓冲的数据。
如果缓冲填满，需要扩展它的空间以便继续读取数据。在`parse_frame()` (未引入)里，
我们需要从`self.buffer[..self.cursor]`这里开始解析数据。

因为与字节数据一起管理位置变量很普遍，`bytes` crate提供了对字节数据和位置变量的抽象封装。
`Buf` 特性就是作为一个能管理数据读取的缓冲类型。而 `BufMut` 特性作为可写数据的缓冲类型。
当传递`T: BufMut`给 `read_buf()`函数时，缓冲的内部位置变量在`read_buf`后被更新。
因此在我们前面的 `read_frame`中不需要自己管理缓冲位置变量。

此外，当使用了 `Vec<u8>`，缓冲需要被初始化。调用`vec![0;4096]` 将分配一个4096字节长的数组，
并被初始化为全是0。当扩展缓冲时，新分配的空间也需要被初始化为0。初始化的动作并非无成本。当
采用`BytesMut` 和 `BufMut`时，缓冲空间没有被初始化。`BytesMut` 抽象会阻止我们读到未初始化
的内存区域。这让我们避免了调用初始化过程。

[`BufMut`]: https://docs.rs/bytes/1/bytes/trait.BufMut.html
[`bytes`]: https://docs.rs/bytes/

# 解析

现在，我们看看 `parse_frame()` 函数。通过两个步骤来解析：

1. 确保已缓冲了完整的帧，帧结尾序号被找到；
2. 解析这个帧；

`mini-redis` crate 提供了实现那两步的函数:

1. [`Frame::check`](https://docs.rs/mini-redis/0.4/mini_redis/frame/enum.Frame.html#method.check)
2. [`Frame::parse`](https://docs.rs/mini-redis/0.4/mini_redis/frame/enum.Frame.html#method.parse)

我们将复用 `Buf` 抽象来提供帮助。`Buf` 被传递给`Frame::check`。在 `check` 函数在传入的缓冲上迭代时，
内部位置指针将被前移。当`check` 返回时，位置指针指向了帧结尾。

对应 `Buf` 类型，我们使用 [`std::io::Cursor<&[u8]>`][`Cursor`].

```rust
use mini_redis::{Frame, Result};
use mini_redis::frame::Error::Incomplete;
use bytes::Buf;
use std::io::Cursor;

# pub struct Connection {
#     stream: tokio::net::TcpStream,
#     buffer: bytes::BytesMut,
# }
# impl Connection {
fn parse_frame(&mut self)
    -> Result<Option<Frame>>
{
    // Create the `T: Buf` type.
    let mut buf = Cursor::new(&self.buffer[..]);

    // Check whether a full frame is available
    match Frame::check(&mut buf) {
        Ok(_) => {
            // Get the byte length of the frame
            let len = buf.position() as usize;

            // Reset the internal cursor for the
            // call to `parse`.
            buf.set_position(0);

            // Parse the frame
            let frame = Frame::parse(&mut buf)?;

            // Discard the frame from the buffer
            self.buffer.advance(len);

            // Return the frame to the caller.
            Ok(Some(frame))
        }
        // Not enough data has been buffered
        Err(Incomplete) => Ok(None),
        // An error was encountered
        Err(e) => Err(e.into()),
    }
}
# }
```

完整的 [`Frame::check`][check] 函数实现能在 [这里][check]找到。我们不涉及它的完整实现。

相关需要注意的是，用到了 `Buf`的 "byte iterator" 风格的API。它们用来读取数据和前移内部
位置指针。例如，为传递帧，检查首字节以确定这个帧的类型。使用的函数是 [`Buf::get_u8`]。
它读取指针位置的字节并前移指针到下一个字节位置。

在[`Buf`] trait里有更多有用的函数。从 [API docs][`Buf`]可以得到更多细节。

[check]: https://github.com/tokio-rs/mini-redis/blob/tutorial/src/frame.rs#L63-L100
[`Buf::get_u8`]: https://docs.rs/bytes/1/bytes/buf/trait.Buf.html#method.get_u8
[`Buf`]: https://docs.rs/bytes/1/bytes/buf/trait.Buf.html
[`Cursor`]: https://doc.rust-lang.org/stable/std/io/struct.Cursor.html

# 有缓冲的写操作

The other half of the framing API is the `write_frame(frame)` function. This
function writes an entire frame to the socket. In order to minimize `write`
syscalls, writes will be buffered. A write buffer is maintained and frames are
encoded to this buffer before being written to the socket. However, unlike
`read_frame()`, the entire frame is not always buffered to a byte array before
writing to the socket.

Consider a bulk stream frame. The value being written is `Frame::Bulk(Bytes)`.
The wire format of a bulk frame is a frame head, which consists of the `$`
character followed by the data length in bytes. The majority of the frame is the
contents of the `Bytes` value. If the data is large, copying it to an
intermediate buffer would be costly.

To implement buffered writes, we will use the [`BufWriter` struct][buf-writer].
This struct is initialized with a `T: AsyncWrite` and implements `AsyncWrite`
itself. When `write` is called on `BufWriter`, the write does not go directly to
the inner writer, but to a buffer. When the buffer is full, the contents are
flushed to the inner writer and the inner buffer is cleared. There are also
optimizations that allow bypassing the buffer in certain cases.

We will not attempt a full implementation of `write_frame()` as part of the
tutorial. See the full implementation [here][write-frame].

First, the `Connection` struct is updated:


```rust
use tokio::io::BufWriter;
use tokio::net::TcpStream;
use bytes::BytesMut;

pub struct Connection {
    stream: BufWriter<TcpStream>,
    buffer: BytesMut,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream: BufWriter::new(stream),
            buffer: BytesMut::with_capacity(4096),
        }
    }
}
```

Next, `write_frame()` is implemented.

```rust
use tokio::io::{self, AsyncWriteExt};
use mini_redis::Frame;

# struct Connection {
#   stream: tokio::io::BufWriter<tokio::net::TcpStream>,
#   buffer: bytes::BytesMut,
# }
# impl Connection {
async fn write_frame(&mut self, frame: &Frame)
    -> io::Result<()>
{
    match frame {
        Frame::Simple(val) => {
            self.stream.write_u8(b'+').await?;
            self.stream.write_all(val.as_bytes()).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Error(val) => {
            self.stream.write_u8(b'-').await?;
            self.stream.write_all(val.as_bytes()).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Integer(val) => {
            self.stream.write_u8(b':').await?;
            self.write_decimal(*val).await?;
        }
        Frame::Null => {
            self.stream.write_all(b"$-1\r\n").await?;
        }
        Frame::Bulk(val) => {
            let len = val.len();

            self.stream.write_u8(b'$').await?;
            self.write_decimal(len as u64).await?;
            self.stream.write_all(val).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Array(_val) => unimplemented!(),
    }

    self.stream.flush().await;

    Ok(())
}
# async fn write_decimal(&mut self, val: u64) -> io::Result<()> { unimplemented!() }
# }
```

The functions used here are provided by [`AsyncWriteExt`]. They are available on
`TcpStream` as well, but it would not be advisable to issue single byte writes
without the intermediate buffer.

* [`write_u8`] writes a single byte to the writer.
* [`write_all`] writes the entire slice to the writer.
* [`write_decimal`] is implemented by mini-redis.

The function ends with a call to `self.stream.flush().await`. Because
`BufWriter` stores writes in an intermediate buffer, calls to `write` do not
guarantee that the data is written to the socket. Before returning, we want the
frame to be written to the socket. The call to `flush()` writes any data pending
in the buffer to the socket.

Another alternative would be to **not** call `flush()` in `write_frame()`.
Instead, provide a `flush()` function on `Connection`. This would allow the
caller to write queue multiple small frames in the write buffer then write them
all to the socket with one `write` syscall. Doing this complicates the
`Connection` API. Simplicity is one of Mini-Redis' goals, so we decided to
include the `flush().await` call in `fn write_frame()`.


[buf-writer]: https://docs.rs/tokio/1/tokio/io/struct.BufWriter.html
[write-frame]: https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs#L159-L184
[`AsyncWriteExt`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html
[`write_u8`]: https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_u8
[`write_decimal`]: https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs#L225-L238

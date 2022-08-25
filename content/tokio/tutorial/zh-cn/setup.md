---
title: "设置"
---

这份指南将带您一步一步理解构建[Redis]客户端和服务端的过程。我将从Rust基本的异步编程开始，继续构建出Redis命令的子集，以得到对Tokio的综合理解。

# Mini-Redis

在这份指南中创建的项目也在 [Mini-Redis on GitHub][mini-redis]. 构建Mini-Redis的目的是学习Tokio, 所以有良好的注释，
但也意味着它缺少一些真正Redis库的功能。可以在[crates.io](https://crates.io/)找到适合生存环境的Redis库。

我们将在例子中直接使用Mini-Redis，这样可以在后续的例子中实现他们之前就用到Mini-Redis的部分功能。

# 帮助

任何地方遇到问题时可以从 [Discord] or [GitHub
discussions][disc]获得帮助。不用担心初学者问题，我们都是从某些地方学起，乐意协助。

[discord]: https://discord.gg/tokio
[disc]: https://github.com/tokio-rs/tokio/discussions

# 前提条件

读者需要基本熟悉Rust。[Rust book][book] 是一个很好的起步学习资源。
非必须，但如果具有有一些Rust标准库[Rust standard library][std] 或其他语言的网络编程经验会更有帮助。
倒不用具备Redis知识。

[rust]: https://rust-lang.org
[book]: https://doc.rust-lang.org/book/
[std]: https://doc.rust-lang.org/std/

## Rust

开始之前，确保已安装[Rust][install-rust] 工具链。如果没有，可很容易使用 [rustup]安装它们。
本指南样例代码的Rust最低版本是 `1.45.0`, 但也推荐您使用最新的稳定版。

使用如下命令检查Rust安装情况:

```bash
$ rustc --version
```

你可以看到类似这样的输出 `rustc 1.46.0 (04488afe3 2020-08-24)`.

## Mini-Redis server

接着安装 Mini-Redis 服务器，可用于测试我们的样例代码。

```bash
$ cargo install mini-redis
```

可以启动它以确保是否正确安装了:

```bash
$ mini-redis-server
```

接着在另一个终端窗口用 `mini-redis-cli`请求键值 `foo`

```bash
$ mini-redis-cli get foo
```

你可以看到 `(nil)`.

# 继续前进

所有的环境都准备好了，进入下一页开始编写您的第一个Rust异步程序。

[redis]: https://redis.io
[mini-redis]: https://github.com/tokio-rs/mini-redis
[install-rust]: https://www.rust-lang.org/tools/install
[rustup]: https://rustup.rs/

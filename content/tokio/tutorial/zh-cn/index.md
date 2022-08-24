---
title: "指南"
subtitle: "概述"
---

Tokio是Rust编程语言中的一个异步运行时框架。它提供了编写网络程序的必要组件模块。它具备运行在包括大型多核服务器到小型嵌入式设备等广泛系统环境的灵活性。 

整体看, Tokio提供主要组件包括:

 - 一个执行异步带宽的多线程运行时.
 - 标准库的一套异步实现.
 - 一个基于编程库的生态.

# Tokio在项目中的角色

当用异步的方式编写应用程序时，你可以让它通过同时执行很多任务的方式获得扩展性和降低成本。
但Rust代码自身并没有异步环境, 需要你来选择一个运行时执行它们。Tokio库就是被最广泛使用的运行时,
远超其他同类库.

此外，Tokio库也提供很多有用的工具类。在编程异步程序时，你没法使用标准库提供的阻塞时API，Tokio库正好提供了相应的替代，
也就是标准库的镜像版本，这很有意义。

# Tokio的优点

本节概述下Tokio的一些优势。

## 执行快

Tokio速度快，是因它基于本身就快Rust语言构建。通过利用语言内嵌特性获得的执行效率很难通过手写代码获得。

Tokio的可扩展, 是基于语言的async/await可扩展特性。当处理网络IO时, 处理一个连接的速度取决于延迟, 
所以提升整体速度唯一的方式就同时处理很多连接。通过async/await特性，增加网络并发操作的代价很低，可以让你执行大量的并发任务。

## 可靠

Tokio基于Rust语言, Rust能让每个人构建可靠、高效的软件。[大量][microsoft]的[研究][chrome] 表明大约~70%高风险安全问题都是
由不安全的内存操作引起。使用Rust可以帮你的应用程序清除这个类别的风险。

Tokio也主要聚焦于提供无意外的一致性的执行行为。它的主要目标是允许用户构建稳定性可预期的软件，以从部署的第一天起就不会在响应请求
方面存在不可预期的延迟尖峰。

[microsoft]: https://www.zdnet.com/article/microsoft-70-percent-of-all-security-bugs-are-memory-safety-issues/
[chrome]: https://www.chromium.org/Home/chromium-security/memory-safety

## 简单

With Rust's async/await feature, the complexity of writing asynchronous
applications has been lowered substantially. Paired with Tokio's utilities and
vibrant ecosystem, writing applications is a breeze.

Tokio follows the standard library's naming convention when it makes sense. This
allows easily converting code written with only the standard library to code
written with Tokio. With the strong type system of Rust, the ability to deliver
correct code easily is unparalleled.

## 灵活性

Tokio provides multiple variations of the runtime. Everything from a
multi-threaded, [work-stealing] runtime to a light-weight, single-threaded
runtime. Each of these runtimes come with many knobs to allow users to tune them
to their needs.

[work-stealing]: https://en.wikipedia.org/wiki/Work_stealing

# When not to use Tokio

Although Tokio is useful for many projects that need to do a lot of things
simultaneously, there are also some use-cases where Tokio is not a good fit.

 - Speeding up CPU-bound computations by running them in parallel on several
   threads. Tokio is designed for IO-bound applications where each individual
   task spends most of its time waiting for IO. If the only thing your
   application does is run computations in parallel, you should be using
   [rayon]. That said, it is still possible to "mix & match"
   if you need to do both.
 - Reading a lot of files. Although it seems like Tokio would be useful for
   projects that simply need to read a lot of files, Tokio provides no advantage
   here compared to an ordinary threadpool. This is because operating systems
   generally do not provide asynchronous file APIs.
 - Sending a single web request. The place where Tokio gives you an advantage is
   when you need to do many things at the same time. If you need to use a
   library intended for asynchronous Rust such as [reqwest], but you don't need
   to do a lot of things at once, you should prefer the blocking version of that
   library, as it will make your project simpler. Using Tokio will still work,
   of course, but provides no real advantage over the blocking API. If the
   library doesn't provide a blocking API, see [the chapter on
   bridging with sync code][bridging].

[rayon]: https://docs.rs/rayon/
[reqwest]: https://docs.rs/reqwest/
[bridging]: /tokio/topics/bridging

# Getting Help

At any point, if you get stuck, you can always get help on [Discord] or [GitHub
discussions][disc]. Don't worry about asking "beginner" questions. We all start
somewhere and are happy to help.

[discord]: https://discord.gg/tokio
[disc]: https://github.com/tokio-rs/tokio/discussions

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

依靠Rust的async/await特性, 编写异步程序的难度有了实质性的降低。结合Tokio的功能组件和活跃的生态，编写异步应用也就极为简单了。

Tokio基本尽可能遵从了标准库的命名风格。只依靠标准版写的代码很容易转换为依靠Tokio库。通过Rust强大的类型系统，就无以伦比地降低了
编写正确代码的难度。 

## 灵活性

Tokio提供了多种形态运行时，包括多线程[工作队列]运行时、轻量单线程运行时。每种运行时都有需要开关方便用户
有需要时自行优化。

[work-stealing]: https://en.wikipedia.org/wiki/Work_stealing

# 何时不适合用Tokio

尽管Tokio对很多需要处理大量并发任务的项目有帮助的，仍有很多场景并不适合用它。

 - 通过多线程并行执行来加速CPU密集型任务. 设计Tokio是为I/O密集型任务应用场景加速，每个任务连接花费大量时间在等待I/O操作上。
　 如果你的应用需要对计算过程进行并行加速，那应用使用[rayon]。如果你同时需要I/O密集型和CPU密集型支持，也有可能进行 "mix & match"使用。
 - 读取很多文件。虽然Tokio好像对读取很多文件的项目看起来有用，但相比一般线程池并没有优势，因为操作系统一般并没有提供异步文件API。
 - 发送单个网络请求。ToKio提供的支持是并发多任务，如果你想要异步直接用其他Rust库如[reqwest],若并不需要大量并发任务执行，
 　采用其阻塞版本的库可以让项目结构更简洁。这时仍可以用Tokio，但并没有任何优势。如果这个库并没有阻塞式API，可以参考这里　[the chapter on
   bridging with sync code][bridging].

[rayon]: https://docs.rs/rayon/
[reqwest]: https://docs.rs/reqwest/
[bridging]: /tokio/topics/bridging

# 获得帮助

任何时候当你遇到使用困难时可以在 [Discord] 或 [GitHub discussions][disc]　得到支持. 不要担心提出初学者问题，我们很高兴协助。

[discord]: https://discord.gg/tokio
[disc]: https://github.com/tokio-rs/tokio/discussions

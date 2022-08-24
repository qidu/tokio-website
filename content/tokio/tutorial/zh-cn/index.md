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

# Advantages of Tokio

This section will outline some advantages of Tokio.

## Fast

Tokio is _fast_, built on top of the Rust programming language, which itself is
fast. This is done in the spirit of Rust with the goal that you should not be
able to improve the performance by writing equivalent code by hand.

Tokio is _scalable_, built on top of the async/await language feature, which
itself is scalable. When dealing with networking, there's a limit to how fast
you can handle a connection due to latency, so the only way to scale is to
handle many connections at once. With the async/await language feature,
increasing the number of concurrent operations becomes incredibly cheap,
allowing you to scale to a large number of concurrent tasks.

## Reliable

Tokio is built using Rust, which is a language that empowers everyone
to build reliable and efficient software. A [number][microsoft] of
[studies][chrome] have found that roughly ~70% of high severity security bugs
are the result of memory unsafety. Using Rust eliminates this entire class of
bugs in your applications.

Tokio also focuses heavily on providing consistent behaviour with no surprises.
Tokio's major goal is to allow users to deploy predictable software that will
perform the same day in and day out with reliable response times and no
unpredictable latency spikes.

[microsoft]: https://www.zdnet.com/article/microsoft-70-percent-of-all-security-bugs-are-memory-safety-issues/
[chrome]: https://www.chromium.org/Home/chromium-security/memory-safety

## Easy

With Rust's async/await feature, the complexity of writing asynchronous
applications has been lowered substantially. Paired with Tokio's utilities and
vibrant ecosystem, writing applications is a breeze.

Tokio follows the standard library's naming convention when it makes sense. This
allows easily converting code written with only the standard library to code
written with Tokio. With the strong type system of Rust, the ability to deliver
correct code easily is unparalleled.

## Flexible

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

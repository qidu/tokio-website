---
title: Getting started with Tracing
---

The [`tracing`] crate is a framework for instrumenting Rust programs to collect
structured, event-based diagnostic information.

In asynchronous systems like Tokio, interpreting traditional log messages can
often be quite challenging. Since individual tasks are multiplexed on the same
thread, associated events and log lines are intermixed making it difficult to
trace the logic flow. `tracing` expands upon logging-style diagnostics by
allowing libraries and applications to record structured events with additional
information about *temporality* and *causality* — unlike a log message, a
[`Span`] in `tracing` has a beginning and end time, may be entered and exited
by the flow of execution, and may exist within a nested tree of similar spans.
For representing things that occur at a single moment in time, `tracing`
provides the complementary concept of *events*. Both [`Span`]s and [`Event`]s
are *structured*, with the ability to record typed data as well as textual
messages.

[`Span`]: https://docs.rs/tracing/latest/tracing/#spans
[`Event`]: https://docs.rs/tracing/latest/tracing/#events

You can use `tracing` to:
- emit distributed traces to an [OpenTelemetry] collector
- debug your application with [Tokio Console]
- log to [`stdout`], [a log file] or [`journald`]
- [profile] where your application is spending time

[`tracing`]: https://docs.rs/tracing
[`tracing-subscriber`]: https://docs.rs/tracing-subscriber
[OpenTelemetry]: https://docs.rs/tracing-opentelemetry
[Tokio Console]: https://docs.rs/console-subscriber
[`stdout`]: https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html
[a log file]: https://docs.rs/tracing-appender/latest/tracing_appender/
[`journald`]: https://docs.rs/tracing-journald/latest/tracing_journald/
[profile]: https://docs.rs/tracing-timing/latest/tracing_timing/

# Setup

To begin, add [`tracing`] and [`tracing-subscriber`] as dependencies:

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = "0.3"
```

The [`tracing`] crate provides the API we will use to emit traces. The
[`tracing-subscriber`] crate provides some basic utilities for forwarding those
traces to external listeners (e.g., `stdout`).

# Subscribing to Traces

If you are authoring an executable (as opposed to a library), you will need to
register a tracing [*subscriber*]. Subscribers are types that process traces
emitted by your application and its dependencies, and and can perform tasks
such as computing metrics, monitoring for errors, and re-emitting traces to the
outside world (e.g., `journald`, `stdout`, or an `open-telemetry` daemon).

[*subscriber*]: https://docs.rs/tracing/latest/tracing/#subscribers

In most circumstances, you should register your tracing subscriber as early as
possible in your `main` function. For instance, the [`FmtSubscriber`] type
provided by [`tracing-subscriber`] prints formatted traces and events to
`stdout`, can be registered like so:

```rust
# mod mini_redis {
#       pub type Error = Box<dyn std::error::Error + Send + Sync>;
#       pub type Result<T> = std::result::Result<T, Error>;
# }
#[tokio::main]
pub async fn main() -> mini_redis::Result<()> {
    // construct a subscriber that prints formatted traces to stdout
    let subscriber = tracing_subscriber::FmtSubscriber::new();
    // use that subscriber to process traces emitted after this point
    tracing::subscriber::set_global_default(subscriber)?;
# /*
    ...
# */ Ok(())
}
```

[`FmtSubscriber`]: https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html

If you run your application now, you may see some trace events emitted by Tokio,
but you will need to modify your own application to emit traces to get the most
out of `tracing`.

##  Subscriber Configuration

In the above example, we've configured [`FmtSubscriber`] with its default
configuration. However, `tracing-subscriber` also provides a number of ways to
configure the [`FmtSubscriber`]'s behavior, such as customizing the output
format, including additional information (such as thread IDs or source code
locations) in the logs, and writing the logs to somewhere other than `stdout`.

For example:
```rust
// Start configuring a `fmt` subscriber
let subscriber = tracing_subscriber::fmt()
    // Use a more compact, abbreviated log format
    .compact()
    // Display source code file paths
    .with_file(true)
    // Display source code line numbers
    .with_line_number(true)
    // Display the thread ID an event was recorded on
    .with_thread_ids(true)
    // Don't display the event's target (module path)
    .with_target(false)
    // Build the subscriber
    .finish();
```

For details on the available configuration options, see [the
`tracing_subscriber::fmt` documentation][fmt-cfg].


In addition to the [`FmtSubscriber`] type from [`tracing-subscriber`], other
`Subscriber`s can implement their own ways of recording `tracing` data. This
includes alternative output formats, analysis and aggregation, and integration
with other systems such as distributed tracing or log aggregation services. A
number of crates provide additional `Subscriber` implementations that may be of
interest. See [here][related-crates] for an (incomplete) list of additional
`Subscriber` implementations.

Finally, in some cases, it may be useful to combine multiple different ways of
recording traces together to build a single `Subscriber` that implements
multiple behaviors. For this purpose, the `tracing-subscriber` crate provides a
[`Layer`] trait that represents a component that may be composed together with
other `Layer`s to form a `Subscriber`. See [here][`Layer`] for details on using
`Layer`s.

[fmt-cfg]: https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html#configuration
[related-crates]: https://docs.rs/tracing/latest/tracing/index.html#related-crates
[`Layer`]: https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/index.html

# Emitting Spans and Events

The easiest way to emit spans is with the [`instrument`] proc-macro annotation
provided by [`tracing`], which re-writes the bodies of functions to emit spans
each time they are invoked; e.g.:

```rust
#[tracing::instrument]
fn trace_me(a: u32, b: u32) -> u32 {
    a + b
}
```

Each invocation of `trace_me` will emit a `tracing` Span that:

1. has a verbosity [level] of `trace` (the greatest verbosity),
2. is named `trace_me`,
3. has fields `a` and `b`, whose values are the arguments of `trace_me`

[`instrument`]: https://docs.rs/tracing/latest/tracing/attr.instrument.html

The [`instrument`] attribute is highly configurable; e.g., to trace
the method in [`mini-redis-server`] that handles each connection:

[`mini-redis-server`]: ../tutorial/setup#mini-redis

```rust
# struct Handler {
#     connection: tokio::net::TcpStream,
# }
# pub type Error = Box<dyn std::error::Error + Send + Sync>;
# pub type Result<T> = std::result::Result<T, Error>;
use tracing::instrument;

impl Handler {
    /// Process a single connection.
    #[instrument(
        level = "info",
        name = "Handler::run",
        skip(self),
        fields(
            // `%` serializes the peer IP addr with `Display`
            peer_addr = %self.connection.peer_addr().unwrap()
        ),
    )]
    async fn run(&mut self) -> mini_redis::Result<()> {
# /*
        ...
# */ Ok::<_, _>(())
    }
}
```

[`mini-redis-server`] will now emit a `tracing` Span for each incoming
connection that:

1. has a verbosity [level] of `info` (the "middle ground" verbosity),
2. is named `Handler::run`,
3. has some structured data associated with it.
    - `fields(...)` indicates that emitted span *should* include
    the `fmt::Display` representation of the connection's `SocketAddr` in a field
    called `peer_addr`.
    - `skip(self)` indicates that emitted span should *not* record `Hander`'s debug representation.

[level]: https://docs.rs/tracing/latest/tracing/struct.Level.html

You can also construct a [`Span`] manually by invoking the [`span!`] macro, or
any of its leveled shorthands ([`error_span!`], [`warn_span!`], [`info_span!`],
[`debug_span!`], [`trace_span!`]).

[`span!`]: https://docs.rs/tracing/*/tracing/macro.span.html
[`error_span!`]: https://docs.rs/tracing/*/tracing/macro.error_span.html
[`warn_span!`]: https://docs.rs/tracing/*/tracing/macro.warn_span.html
[`info_span!`]: https://docs.rs/tracing/*/tracing/macro.info_span.html
[`debug_span!`]: https://docs.rs/tracing/*/tracing/macro.debug_span.html
[`trace_span!`]: https://docs.rs/tracing/*/tracing/macro.trace_span.html

To emit events, invoke the [`event!`] macro, or any of its leveled shorthands
([`error!`], [`warn!`], [`info!`], [`debug!`], [`trace!`]). For instance, to
log that a client sent a malformed command:
```rust
# type Error = Box<dyn std::error::Error + Send + Sync>;
# type Result<T> = std::result::Result<T, Error>;
# struct Command;
# impl Command {
#     fn from_frame<T>(frame: T) -> Result<()> {
#         Result::Ok(())
#     }
#     fn from_error<T>(err: T) {}
# }
# let frame = ();
// Convert the redis frame into a command struct. This returns an
// error if the frame is not a valid redis command.
let cmd = match Command::from_frame(frame) {
    Ok(cmd) => cmd,
    Err(cause) => {
        // The frame was malformed and could not be parsed. This is
        // probably indicative of an issue with the client (as opposed
        // to our server), so we (1) emit a warning...
        //
        // The syntax here is a shorthand provided by the `tracing`
        // crate. It can be thought of as similar to:
        //      tracing::warn! {
        //          cause = format!("{}", cause),
        //          "failed to parse command from frame"
        //      };
        // `tracing` provides structured logging, so information is
        // "logged" as key-value pairs.
        tracing::warn! {
            %cause,
            "failed to parse command from frame"
        };
        // ...and (2) respond to the client with the error:
        Command::from_error(cause)
    }
};
```

[`event!`]: https://docs.rs/tracing/*/tracing/macro.event.html
[`error!`]: https://docs.rs/tracing/*/tracing/macro.error.html
[`warn!`]: https://docs.rs/tracing/*/tracing/macro.warn.html
[`info!`]: https://docs.rs/tracing/*/tracing/macro.info.html
[`debug!`]: https://docs.rs/tracing/*/tracing/macro.debug.html
[`trace!`]: https://docs.rs/tracing/*/tracing/macro.trace.html

If you run your application, you now will see events decorated with their span's
context emitted for each incoming connection that it processes.

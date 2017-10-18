# Summary
[summary]: #summary

This RFC proposes to simplify and focus the Tokio project, in an attempt to make
it easier to learn and more productive to use. Specifically:

* Add a *default* global event loop, eliminating the need for setting up and
  managing your own event loop in the vast majority of cases.

  * Moreover, remove the distinction between `Handle` and `Remote` by making
  `Handle` both `Send` and `Sync` and deprecating `Remote`. Thus, even working
  with custom event loops becomes simpler.

  * Allow swapping out this default event loop for those who want to exercise
    full control.

* Decouple all task execution functionality from Tokio, instead providing it
  through a standard futures component.

  * When running tasks thread-locally (for non-`Send` futures),
    provide more fool-proof APIs that help avoid lost wakeups.

  * Eventually provide some default thread pools as well (out of scope for this
    RFC).

* Similarly, decouple timer futures from Tokio, providing functionality instead
  through a new `futures-timer` crate.

* Provide the above changes in a new `tokio` crate, which is a slimmed down
  version of today's `tokio-core`, and may *eventually* re-export the contents
  of `tokio-io`. The `tokio-core` crate is deprecated, but will remain available
  for backward compatibility. In the long run, most users should only need to
  depend on `tokio` to use the Tokio stack.

* Focus documentation primarily on `tokio`, rather than on
  `tokio-proto`. Provide a much more extensive set of cookbook-style examples
  and general guidelines, as well as a more in-depth guide to working with
  futures.

Altogether, these changes, together with [async/await], should go a long
distance toward making Tokio a newcomer-friendly library.

[async/await]: https://internals.rust-lang.org/t/help-test-async-await-generators-coroutines/5835

# Motivation
[motivation]: #motivation

While Tokio has been an important step forward in Rust's async I/O story, many
have found it difficult to learn initially, and have wanted more guidance in
using it even after mounting the initial learning curve. While documentation is
a major culprit, there are also several aspects of the core APIs that can be
made simpler and less error-prone. The core motivation of this RFC is to tackle
this problem head-on, ensuring that the technical foundation itself of Tokio is
simplified to enable a much smoother introductory experience into the "world of
async" in Rust.

On the documentation side, one mistake we made early on in the Tokio project was
to so prominently discuss the `tokio-proto` crate in the documentation. While
the crate was intended to make it very easy to get basic request/response protocol
implementations up and running, it did not provide a very useful *entry* point
for learning Tokio as a whole. Moreover, it turns out that there's a much wider
audience for Tokio than there is for `tokio-proto` in particular. It's not
entirely clear what the long-term story for `tokio-proto` should be (given that
[`h2`], one of its intended uses, rolled it own implementation), but for the
time being it will be de-emphasized in the documentation.

[`h2`]: https://github.com/carllerche/h2

On the API side, we've had more success with the `tokio-core` crate, but it's
not without its problems. The distinction between `Core`, `Handle`, and `Remote`
is subtle and can be difficult to grasp, for example. And in general, Tokio
requires some amount of setup to use properly, which has turned out to be a
struggle for some. Simplifying and decoupling these APIs should make the library
easier to learn and to use.

It is our intention that after this reorganization happens the introduction to
the Tokio project is a much more direct and smoother path than it is today.
There will be fewer crates to consider (mostly just `tokio`) which have a
much smaller API to work with (detailed below) and should allow us to tackle
the heart of async programming, futures, much more quickly in the
documentation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## API changes

*Note: this is a guide-level explanation to the *changes*, not a full-blown
attempt to teach Tokio from scratch with the new APIs*.

### A quick overview of the changes

From a high level, the changes are:

- Make it possible to use Tokio without ever explicitly touching an event loop.

- Simplify the APIs around event loops.

- Decouple all task execution functionality from Tokio, moving it to the futures
  level.

- Publish this all as a new `tokio` crate, deprecating `tokio-core` (which just
  wraps `tokio` and can interoperate).

### Examples for common-case usage

To see the API changes at a high level, let's start with a very simple example:
an echo server written directly against Tokio. We'll look, in fact, at *four*
variants of the echo server:

- A blocking variant using `std`.
- A variant using today's `tokio-core`.
- A variant using the proposed `tokio` crate.
- A variant using `tokio` and [`async/await`] notation.

#### Variant 1: a blocking echo server

Here's one way you might write a blocking echo server against `std`:

```rust
//! A blocking echo server

use std::io;
use std::env;
use std::net::{SocketAddr, TcpListener};
use std::thread;

fn serve(addr: SocketAddr) -> io::Result<()> {
    let socket = TcpListener::bind(&addr)?;
    println!("Listening on: {}", addr);

    for conn in socket.incoming() {
        if let Ok(stream) = conn {
            thread::spawn(move || {
                // the strange double-borrow here is needed because
                // `copy` takes `&mut` references
                match io::copy(&mut &stream, &mut &stream) {
                    Ok(amt) => println!("wrote {} bytes to {}", amt, addr),
                    Err(e) => println!("error on {}: {}", addr, e),
                }
            });
        }
    }

    Ok(())
}

fn main() {
    let addr = env::args()
        .nth(1)
        .unwrap_or("127.0.0.1:8080".to_string())
        .parse()
        .unwrap();

    serve(addr).unwrap();
}
```

This code is almost entirely straightforward; the only hiccup is around the
`copy` combinator, which wants `&mut` reference for both a reader and a
writer. Fortunately, a `&TcpStream` can serve as *both* a reader and writer
(thread safety is provided by the operating system).

#### Variant 2: a `tokio-core` echo server

Now we'll translate the blocking code into async code using today's `tokio-core`
crate, trying to keep it as similar as possible:

```rust
#![feature(conservative_impl_trait)]

extern crate futures;
extern crate tokio_core;
extern crate tokio_io;

use std::io;
use std::env;
use std::net::SocketAddr;

use futures::{Future, Stream, IntoFuture};
use tokio_core::net::TcpListener;
use tokio_core::reactor::{Core, Handle};
use tokio_io::AsyncRead;
use tokio_io::io::copy;

fn serve(addr: SocketAddr, handle: Handle) -> impl Future<Item = (), Error = io::Error> {
    TcpListener::bind(&addr, &handle)
        .into_future()
        .and_then(move |socket| {
            println!("Listening on: {}", addr);

            socket
                .incoming()
                .for_each(move |(conn, _)| {
                    let (reader, writer) = conn.split();
                    handle.spawn(copy(reader, writer).then(move |result| {
                        match result {
                            Ok((amt, _, _)) => println!("wrote {} bytes to {}", amt, addr),
                            Err(e) => println!("error on {}: {}", addr, e),
                        };
                        Ok(())
                    }));
                    Ok(())
                })
        })
}

fn main() {
    let addr = env::args()
        .nth(1)
        .unwrap_or("127.0.0.1:8080".to_string())
        .parse()
        .unwrap();

    let mut core = Core::new().unwrap();
    let handle = core.handle();
    core.run(serve(addr, handle)).unwrap();
}
```

Several aspects needed to change to get this variant to work:

- Instead of writing code that primarily works with `io::Result`, we're now
  working with `impl Future` and combinators thereof.

- We need to set up an event loop (`Core`), generate handles to it, and pass
  those handles around.
  - The handles are used both for setting up I/O objects, and for spawning tasks.

- The `Incoming` stream and result of `copy` are both more complex than those in
  `std`.
  - For the `Incoming` stream, that's a minor detail having to do with how the
    socket address is exposed.
  - For `copy`, that's a deeper shift: the function takes full *ownership* of
    the reader and writer, because in general futures avoid borrowing. Instead,
    ownership of the reader and writer is returned upon completion (and here,
    thrown away).

All of these changes take some work to get your head around. Some of them are
due to Tokio's API design; others are due to futures. The next two variants will
show how we can improve each of those two sides.

#### Variant 3: a `tokio` echo server

First, we'll convert the code to use the newly-proposed `tokio` crate:

```rust
#![feature(conservative_impl_trait)]

extern crate futures;
extern crate tokio_core;
extern crate tokio_io;

use std::io;
use std::env;
use std::net::SocketAddr;

use futures::{Future, Stream, IntoFuture};
use futures::thread;
use tokio_core::net::TcpListener;
use tokio_io::AsyncRead;
use tokio_io::io::copy;

fn serve(addr: SocketAddr) -> impl Future<Item = (), Error = io::Error> {
    TcpListener::bind(&addr)
        .into_future()
        .and_then(move |socket| {
            println!("Listening on: {}", addr);

            socket
                .incoming()
                .for_each(move |(conn, _)| {
                    let (reader, writer) = conn.split();
                    thread::spawn_task(copy(reader, writer).then(move |result| {
                        match result {
                            Ok((amt, _, _)) => println!("wrote {} bytes to {}", amt, addr),
                            Err(e) => println!("error on {}: {}", addr, e),
                        };
                        Ok(())
                    }));
                    Ok(())
                })
        })
}

fn main() {
    let addr = env::args()
        .nth(1)
        .unwrap_or("127.0.0.1:8080".to_string())
        .parse()
        .unwrap();

    thread::block_on_all(serve(addr));
}
```

This version of the code retains the move to using futures explicitly, but
eliminates the explicit event loop setup and handle passing. To execute and
spawn tasks, it uses the `thread` executor from futures, which
multiplexes tasks onto the current OS thread. A similar API is available for
working with the default thread pool instead, which is more appropriate for
tasks performing blocking or CPU-heavy work.

#### Variant 4: a `tokio` + `async/await` echo server -- the long-term vision

Finally, if we incorporate [`async/await`] notation, we get code that looks
quite similar to the initial blocking variant:

```rust
#![feature(proc_macro, conservative_impl_trait, generators)]

extern crate futures_await as futures;
extern crate tokio;
extern crate tokio_io;

use std::io;
use std::env;
use std::net::SocketAddr;

use futures::prelude::*;
use futures::thread;
use tokio::net::TcpListener;
use tokio_io::AsyncRead;
use tokio_io::io::copy;

#[async]
fn serve(addr: SocketAddr) -> io::Result<()> {
    let socket = TcpListener::bind(&addr)?;
    println!("Listening on: {}", addr);

    #[async]
    for (conn, _) in socket.incoming() {
        // with Tokio, read and write components are distinct:
        let (reader, writer) = conn.split();

        thread::spawn_task(async_block! {
            match await!(copy(reader, writer)) {
                Ok((amt, _, _)) => println!("wrote {} bytes to {}", amt, addr),
                Err(e) => println!("error on {}: {}", addr, e),
            };
            Ok(())
        });
    }
    Ok(())
}

fn main() {
    let addr = env::args()
        .nth(1)
        .unwrap_or("127.0.0.1:8080".to_string())
        .parse()
        .unwrap();

    thread::block_on_all(serve(&cont, addr));
}
```

The most important difference, of course, is the sprinkling of `async` and
`await` annotations. You can think of them as follows:

- Code that is `async`-annotated looks like `impl Future<Item = T, Error = E>`
  on the outside, and `Result<T, E>` on the inside. That is, it's lets you write
  code that *looks* like synchronous I/O code, but in fact produces a future.
- Code that is `await`-annotated is the reverse: it looks like
  `impl Future<Item = T, Error = E>` on the *inside*, and `Result<T, E>` on the
  outside. You use `await` to consume futures within `async` code.

More detail is available [here](https://github.com/alexcrichton/futures-await);
the rest of this RFC is focused purely on the Tokio side of things.

The *long-term* goal is to provide a complete async story that feels very close to
synchronous programming, i.e. something like:

```rust
async fn serve(addr: SocketAddr) -> io::Result<()> {
    let socket = TcpListener::bind(&addr)?;
    println!("Listening on: {}", addr);

    async for (conn, _) in socket.incoming() {
        // with Tokio, read and write components are distinct:
        let (reader, writer) = conn.split();

        thread::spawn_task(async {
            match await copy(reader, writer) {
                Ok((amt, _, _)) => println!("wrote {} bytes to {}", amt, addr),
                Err(e) => println!("error on {}: {}", addr, e),
            };
            Ok(())
        });
    }

    Ok(())
}
```

However, to fully get there, we'll need *borrowing* to work seamlessly with
`async`/`await`, and for the feature to be provided in a direct way by the
compiler; this is still a ways off. In the meantime, though, we can improve the
Tokio side of the experience, which is what this RFC aims to do.

### Key insights for this RFC

The design in this RFC stems from several key insights, which we'll work through
next.

#### Insight 1: we can decouple task execution from the event loop

In today's `tokio-core` design, the event loop (`Core`) provides two important
pieces of functionality:

- It manages asynchronous I/O events using OS APIs, recording those events and
  waking up tasks as appropriate.

- It acts as an *executor*, i.e. you can spawn futures onto it.

On each turn of the event loop, the `Core` processes all I/O events *and* polls
any ready-to-run futures it's responsible for.

Running tasks directly on the event loop thread is a double-edged sword. On the
one hand, you avoid cross-thread communication when such tasks are woken from
I/O events, which can be advantageous when the task is just going to immediately
set some new I/O in motion. On the other hand, if the woken task does
non-trivial work (even just to determine what I/O to do), *it is effectively
blocking the further processing and dispatch of I/O events*, which can limit the
potential for parallelism.

An insight behind this RFC is that there's no need to *couple* executor
functionality with the event loop; we can provide each independently, but still
allow you to combine them onto a single thread if needed. Our hypothesis is
that, for most users, keeping event processing and task execution on strictly
separate threads is a net win: it eliminates the risk of blocking the event loop
and hence increases parallelism, at a negligible synchronization cost.

Thus, this RFC proposes that the new `tokio` crate focus *solely* on I/O,
leaving executors to the `futures` crate.

#### Insight 2: we can provide default event loops and executors for the common case

Pushing further along the above lines, we can smooth the path toward a
particular setup that we believe will work well for the majority of
applications:

- One dedicated event loop thread, which does not perform any task execution.
- A global thread pool for executing compute-intensive or blocking tasks; tasks must be `Send`.
- Optionally, a dedicated thread for running thread-local (non-`Send`) tasks in
  a *cooperative* fashion. (Requires that the tasks do not perform too much work
  in a given step).

The idea is to provide APIs for (1) working with I/O objects without an explicit
event loop and (2) spawning `Send` tasks without an explicit thread pool, and in
both cases spin up the corresponding default global resource. We then layer APIs
for customizing or configuring the global resource, as well as APIs for
e.g. targeting specific event loops. But for most users, it's just not necessary
to think about the event loop or thread pool.

These default global resources greatly aid the experience for two additional cases:

- Asynchronous client-side code, where it's far less natural to spin up and
  manage an event loop yourself.

- *Synchronous* wrappers around async libraries. These wrappers generally want
  to avoid exposing details about the event loop, which is an implementation
  detail. By providing a global event loop resource, these wrapper libraries can
  both avoid this exposure *and* successfully *share* a common event loop,
  without any action on the user's part.

For thread-local tasks, which are more specialized *and local*, the user should
retain full control over the thread of execution, and specifically request
spinning up an executor. An API for doing so, with some additional fool-proofing
compared to today's APIs, is part of the RFC.

#### Insight 3: we can remove the `Handle`/`Remote` distinction

Today, `tokio-core` provides *three* types for working with event loops:

- `Core`: the event loop itself; non-`Clone`, non-`Send`.
- `Handle`: an *on-thread* pointer to the event loop; `Clone`, non-`Send`.
- `Remote`: a general-purpose pointer to the event loop; `Clone` and `Send`.

The distinction between `Handle` and `Remote` today stems from the fact that the
event loop serves as an executor; to spawn non-`Send` tasks onto it, you need
"proof" that you're on the same thread, which is what `Handle` provides. By
splitting off the executor functionality, we can do away with this distinction.

At the same time, we can incorporate recent changes to `mio` that make it
possible to provide highly-concurrent updates to the event loop itself, so that
registering events cross-thread requires only lightweight synchronization.

#### Insight 4: the `tokio` crate

As we'll see, some of the changes proposed in this RFC would involve breakage
for `tokio-core`. Rather than produce a new major release, however, the RFC
proposes to introduce a `tokio` crate providing the new APIs, which can be
wrapped by the (now deprecated) `tokio-core` crate. This also allows us to
provide interoperation across libraries.

The Tokio team initially resisted creating a `tokio` crate because it was
unclear at first what the most common use-cases of the library would be,
e.g. how important `tokio-proto` would be. It's now clear that today's
`tokio-core` is really what *Tokio* is, with `tokio-proto` being a convenient
helper that sits on top.

Moreover, with the other slimming down proposed in this RFC, the remaining APIs
seem constitute a relatively minimal commitment.

Thus, the time seems ripe to publish a `tokio` crate!

## Documentation changes

This RFC won't go into extensive detail on the new structure of the
documentation, but rather just enumerate some basic constraints:

- It should put much more focus on core futures and Tokio concepts, rather than
  `tokio-proto`, at the outset. This should include talking much more about the
  *ecosystem*, showing how to use existing libraries to build async services.

- It should give far more guidance on how to effectively *use* Tokio and
  futures, e.g. by talking about how to deal with common ownership issues.

- It should provide an order of magnitude more examples, with a mix of
  "cookbook"-style snippets and larger case studies.

- There should be a lot more of it.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The new `tokio` crate

To begin with, the new `tokio` crate will look like a streamlined version of
`tokio-core`. Eventually it will also re-export APIs from `tokio-io`, once those
APIs have been fully vetted.

At the top level, the crate will provide three submodules:

- `reactor`: for manual configuration and control over event loops.
- `net`: I/O objects for async networking.
- `io`: general utilities for async I/O (much like `std::io`)

Let's look at each in turn.

### The `reactor` module

The reactor module provides just a few types for working with reactors (aka
event loops):

- `Reactor`, an owned reactor. (Used to be `Core`)
- `Handle`, a shared, cloneable handle to a reactor.
- `PollEvented`, a bridge between `mio` event sources and a reactor

Compared to `Core` today, the `Reactor` API is much the same, but drops the
`run` and `remote` methods and the `Executor` implementation:

```rust
impl Reactor {
    pub fn new() -> Result<Reactor>;
    pub fn handle(&self) -> Handle;

    // blocks, turning the event loop until either woken through a handle,
    // or the given timeout expires.
    pub fn turn(&mut self, max_wait: Option<Duration>) -> Turn;
}

// No contents/API initially, but we may want to expand this in the future
pub struct Turn { /* ... */ }
```

The `Handle` API is also slimmed down:

```rust
impl Default for Handle {
    // get a handle to the default global event loop ...
}

impl Handle {
    // Wakes up the corresponding reactor if it is blocking
    pub fn wakeup(&self);

    // Sets this handle as the default (returned by `Handle::default`)
    // within the current thread, for the duration of `Enter`'s lifetime.
    //
    // The `Enter` type is explained in a later section on executors,
    // but the point here is that defaults are tied to executor
    // granularity.
    pub fn make_default_for(&self, &mut Enter);
}
```

Most usage of the `Handle` type comes through *parameters* to various I/O object
construction functions, as we'll see in a moment.

Unlike today, though, `Handle` is `Send` and `Sync` (as well as `Clone`).
Handles are used solely to tie particular I/O objects to particular event loops.
In particular, the `PollEvented` API stays exactly as-is, and is the primitive
way that I/O objects are registered onto a reactor.

### The `net` module

The `net` module remains almost exactly as it is in `tokio-core` today, with one
key difference: APIs for constructing I/O objects come in two flavors, one that
uses the default global reactor, and one that takes an explicit
handle. Generally the convenient, "default" mode of construction doesn't take a
handle, and those involving customizations do.

#### Example: `TcpListener`

```rust
impl TcpListener {
    // set up a listener with the global event loop
    fn bind(addr: &SocketAddr) -> io::Result<TcpListener>;

    // set up a fully-customized listener
    fn from_std(listener: std::net::TcpListener, handle: &Handle) -> io::Result<TcpListener>;

    // this yields a *std* TcpStream, so that you can use `TcpStream::from_std` to
    // associate it with an handle of your choice.
    fn accept_std(&mut self) -> Result<(std::net::TcpStream, SocketAddr)>

    // by contrast, this method returns a stream bound to the default handle.
    fn accept(&mut self) -> Result<(TcpStream, SocketAddr)>
}
```

#### Example: `TcpStream`

```rust
impl TcpStream {
    // set up a stream with the global event loop
    fn connect(addr: &SocketAddr) -> TcpStreamNew;

    // these are as today
    fn from_std(stream: std::net::TcpStream, handle: &Handle) -> Result<TcpStream>;
    fn connect_std(stream: std::net::TcpStream, addr: &SocketAddr, handle: &Handle) -> TcpStreamNew
}
```

### The `io` module

Finally, there may *eventually* be an `io` module with the some portion of
contents of the `tokio-io` crate. However, those APIs need a round of scrutiny
first (the subject of a future RFC), and they may ultimately be moved into the
`futures` crate instead.

An eventual goal, in any case, is that you can put Tokio to good use by bringing
in only the `tokio` and `futures` crates.

## Changes to the `futures` crate

In general, futures are spawned onto *executors*, which are responsible for
polling them to completion. There are basically two ways this can work: either
you run a future on your thread, or someone else's (i.e. a thread pool).

The threadpool case is provided by crates like [futures_cpupool]; eventually, a
crate along these lines should provide a configurable, default global thread
pool, which provides similar benefits to providing a global event loop. This
functionality may eventually want to live in the `futures` crate. Exact details
are out of scope for this RFC.

[futures_cpupool]: https://docs.rs/futures-cpupool/0.1.5/futures_cpupool/

The "your thread" case allows for non-`Send` futures to be scheduled
cooperatively onto their thread of origin, and is *currently* provided by the
`Core` in `tokio_core`. As explained in the Guide-level introduction, though,
this RFC completely decouples this functionality, moving it instead into the
`futures` crate.

### The `thread` module

The `futures` crate will add a module, `thread`, with the following
contents:

```rust
pub mod thread {
    /// Execute the given future *synchronously* on the current thread, blocking until
    /// it (and all spawned tasks) completes and returning its result.
    ///
    /// In more detail, this function blocks until:
    /// - the given future completes, *and*
    /// - all spawned tasks complete, or `cancel_all_spawned` is invoked
    ///
    /// Note that there is no `'static` or `Send` requirement on the future.
    pub fn block_on_all<F: Future>(f: F)  -> Result<F::Item, F::Error>;

    /// Execute the given closure, then block until all spawned tasks complete.
    ///
    /// In more detail, this function will block until:
    /// - All spawned tasks are complete, or
    /// - `cancel_all_spawned` is invoked.
    pub fn block_with_init<F>(f: F) where F: FnOnce(&Enter);

    /// Spawns a task, i.e. one that must be explicitly either
    /// blocked on or killed off before `block_*` will return.
    ///
    /// # Panics
    ///
    /// This function can only be invoked within a future given to a `block_*`
    /// invocation; any other use will result in a panic.
    pub fn spawn_task<F>(task: F) where F: Future<Item = (), Error = ()> + 'static;

    /// Spawns a daemon, which does *not* block the pending `block_on_all` call.
    ///
    /// # Panics
    ///
    /// This function can only be invoked within a future given to a `block_*`
    /// invocation; any other use will result in a panic.
    pub fn spawn_daemon<F>(task: F) where F: Future<Item = (), Error = ()> + 'static;

    /// Cancels *all* spawned tasks and daemons.
    ///
    /// # Panics
    ///
    /// This function can only be invoked within a future given to a `block_*`
    /// invocation; any other use will result in a panic.
    pub fn cancel_all_spawned(&self);

    struct TaskExecutor { .. }
    impl<F> Executor<F> for TaskExecutor
        where F: Future<Item = (), Error = ()> + 'static { .. }

    /// Provides an executor handle for spawning tasks onto the current thread.
    ///
    /// # Panics
    ///
    /// As with the `spawn_*` functions, this function can only be invoked within
    /// a future given to a `block_*` invocation; any other use will result in
    // a panic.
    pub fn task_executor() -> TaskExecutor;

    struct DaemonExecutor { .. }
    impl<F> Executor<F> for DaemonExecutor
        where F: Future<Item = (), Error = ()> + 'static { .. }

    /// Provides an executor handle for spawning daemons onto the current thread.
    ///
    /// # Panics
    ///
    /// As with the `spawn_*` functions, this function can only be invoked within
    /// a future given to a `block_*` invocation; any other use will result in
    // a panic.
    pub fn daemon_executor() -> DaemonExecutor;
}
```

This suite of functionality replaces two APIs provided in the Tokio stack today:

- The free `block_on_all` function replaces the `wait` method on `Future`, which
  will be deprecated. In our experience with Tokio, as well as the experiences
  of other ecosystems like Finagle in Scala, having a blocking method so easily
  within reach on futures leads people down the wrong path. While it's vitally
  important to have this "bridge" between the async and sync worlds, providing
  it as `thread::block_on_all` highlights the synchronous nature. In addition,
  the fact that the function automatically blocks on any spawned tasks helps
  avoid footguns as well.

- Today, Tokio's `Handle` can be used for cooperative, non-`Send` task
  execution. Here, that functionality is replaced (and enhanced) by a suite of
  free functions, `spawn_task`, `spawn_daemon`, and `cancel_all_spawned`. These
  functions, which can only be called in the context of a `block_*` function,
  give you ways both to spawn cooperative tasks, and to fully manage
  shutdown. In particular, the fact that `block_on_all` waits for spawned tasks
  to complete (or for explicit cancellation) helps mitigate another footgun with
  today's setup: tasks that are dropped on the floor.

Those wishing to couple a reactor and a single-threaded executor, as today's
`tokio-core` does, should use something like `FuturesUnordered` together with a
custom reactor to do so. We expect there will eventually be a separate crate
providing this functionality in a high-performance way.

### Executors in general

Another footgun we want to watch out for is accidentally trying to spin up
multiple executors on a single thread. The most common case is using
`thread::block_on_all` on, say, a thread pool's worker thread.

We can mitigate this easily by providing an API for "executor binding" that
amounts to a thread-local boolean:

```rust
pub mod executor {
    pub struct Enter { .. }

    // Marks the current thread as being within the dynamic extent of
    // an executor. Panics if the current thread is *already* marked.
    pub fn enter() -> Enter;

    impl Enter {
        // Register a callback to be invoked if and when the thread
        // ceased to act as an executor.
        pub fn on_exit<F>(&self, f: F) where F: FnOnce() + 'static;

        // Treat the remainder of execution on this thread as part of an
        // executor; used mostly for thread pool worker threads.
        //
        // All registered `on_exit` callbacks are *dropped* without being
        // invoked.
        pub fn make_permanent(self);
    }

    impl Drop for Enter {
        // Exits the dynamic extent of an executor, unbinding the thread
    }
}
```

This API should be used in functions like `block_on_all` that enter executors,
i.e. that block until some future completes (or other event
occurs). Consequently, nested uses of `block_on_all`, or uses within a thread
pool, will panic, alerting the user to a bug.

Executors should publicly expose access to an `Enter` only during
initialization, e.g. many executors provide for "initialization hooks" for
setting up worker threads, and these hooks and provide temporary access to an
`Enter`. That restriction is leveraged by methods like
`Handle::make_default_for`, which are intended to only be used during executor
thread initialization.

## The `futures-timer` crate

The `futures-timer` crate will contain the `Delay` (used to be `Timeout`) and `Interval` types
currently in `tokio_core`'s `reactor` module:

```rust
impl Delay {
    fn new(dur: Duration) -> Result<Delay>
    fn new_at(at: Instant) -> Result<Delay>
    fn reset(&mut self, at: Instant);
}

impl Future for Delay { .. }

impl Interval {
    fn new(dur: Duration) -> Result<Interval>
    fn new_at(at: Instant, dur: Duration) -> Result<Interval>
    fn reset(&mut self, at: Instant);
}

impl Stream for Interval { .. }
```

These functions will lazily initialize a dedicated timer thread. We will eventually
provide a means of customization, similar to `SetDefault` for reactor, but that's
out of scope for this RFC.

## Migration Guide

While `tokio` and `tokio-core` will provide some level of interoperation, the
expectation is that over time the ecosystem will converge on just using `tokio`.
Transitioning APIs over is largely straightforward, but will require a major
version bump:

- Uses of `Remote`/`Handle` that don't involve spawning tasks can use the new
  `Handle`.
- Local task spawning should migrate to use the `thread` module.
- APIs that need to construct I/O objects, or otherwise require a `Handle`,
  should follow the pattern laid out in the `net` module:
  - There should be a simple standard API that does not take an explicit
    `Handle` argument, instead using the global one.
  - Alongside, there should be some means of customizing the handle, either
    through a secondary API (`connect` and `connect_handle`), or else as part of
    a builder-style API (if there are other customizations that you want to
    allow).

## The story for other Tokio crates

### `tokio-core`

This crate is deprecated in favor of the new `tokio` crate. To provide for
interoperation, however, it is also *reimplemented* as a thin layer on top of
the new `tokio` crate, with the ability to dig into this layering. This
reimplementation will be provided as a new, semver-compatible release.

The key idea is that a `tokio_core` `Core` is a *combination* of a `Reactor` and
a custom thread-local executor, and you can thus extract out the underlying pieces:

```rust
// in the new tokio_core release:

use tokio::reactor;
use futures::thread;

impl Core {
    fn from_tokio_reactor(reactor: reactor::Reactor) -> Self;
    fn tokio_reactor(&mut self) -> &mut reactor::Reactor;
}

impl Handle {
    fn tokio_handle(&self) -> &reactor::Handle;
}
```

This should allow at least some level of interoperation between libraries using
the current `tokio_core` libraries and those that have migrated to `tokio`.

### `tokio-io`

The `tokio-io` crate needs an overall API audit. Its APIs may eventually be
reexported within the `tokio` crate, but it also has value as a standalone
crate that provides general I/O utilities for futures. (It may move into
`futures` instead).

### `tokio-service`

The `tokio-service` crate is not, in fact, Tokio-specific; it provides a general
way of talking about services and middleware in the futures world. Thus the
likely path is for `tokio-service` to be split out from the Tokio project and
instead provide foundational traits under a new moniker. Regardless, though, the
crate will always remain available in some form.

### `tokio-proto`

Finally, there's `tokio-proto`, which provides a very general framework for
Tokio-based servers and clients.

Part of the original goal was for `tokio-proto` to be expressive enough that you
could build a solid http2 implementation with it; this has not panned out so
far, though it's possible that the crate could be improved to do so.  On the
other hand, it's not clear that it's useful to provide that level of
expressiveness in a general-purpose framework.

It seems like there are three basic paths forward:

- Improve `tokio-proto` such that is can support the full `h2`
  implementation. It's not clear to what extent doing so would benefit *other*
  protocols, however.
- Refocus `tokio-proto` on simpler use-cases and ergonomics, e.g. by removing
  streaming bodies entirely.
- Deprecate it, in favor of direct, custom server implementations.

It's also not clear what the long-term story for Hyper is; it currently uses
`tokio-proto`, but may end up using `h2` instead.

So, in general: the future direction for `tokio-proto` is unclear. We need to be
driven by strong, concrete use-cases. If you have one of these, we'd love to
hear about it!

# Drawbacks
[drawbacks]: #drawbacks

The clear drawback here is ecosystem churn. While the proposal provides
interoperation with the existing Tokio ecosystem, there's still a push toward
major version bumps across the ecosystem to adapt to the new idioms. The interop
means, however, that it's far less problematic for an app to "mix" the two
worlds, which means that major version bumps don't need to be as coordinated as
they might otherwise be.

# Rationale and Alternatives
[alternatives]: #alternatives

The rationale has been discussed pretty extensively throughout. However, a few
aspects of the design have quite plausible alternatives:

- The approach to `Handle`s means that libraries need to provide *two* API
  surfaces: one using the default global reactor, and one allowing for
  customization. In practice, it seems likely that many libraries will skip out
  on the second API. Clients that require it will then need to file issues, and
  adding support for handle specification after the fact can involve some tricky
  refactoring -- especially because there's no simple way to ensure that you
  aren't calling some *other* API that's using the default reactor, and should
  *now* be passing a customized one.
  - An alternative would be to use thread-local storage to set the "current
    reactor" in a scoped way. By default, this would be the global reactor, but
    you could override it in cases where you need to customize the reactor
    that's used.
  - Unfortunately, though, this alternative API is subject to some very subtle
    bugs when working with futures, because you need to ensure that this
    thread-local value is set *at the right time*, which means you need to know
    whether the handle is used when *constructing* the future, or when
    *executing* it. It was unclear how to design an API that would make it easy
    to get this right.

# Unresolved questions
[unresolved]: #unresolved-questions

- What should happen with `tokio-io`?
- What are the right use-cases to focus on for `tokio-proto`?

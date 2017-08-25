# Summary
[summary]: #summary

Drastically simplify the Tokio project by addressing some of the major pain
points of using its apis today:

* Remove the distinction between `Handle` and `Remote` in `tokio-core` by making
  `Handle` both `Send` and `Sync`.
* Add a global event loop in `tokio-core` that is managed automatically, along
  with the ability to acquire a global `Handle` reference.
* Focus documentation on `tokio-core` rather than `tokio-proto`, and delegate
  the functionality of `tokio-proto` to upstream projects rather than under the
  umbrella 'Tokio' moniker.



# Motivation
[motivation]: #motivation

Perhaps the largest roadblock to Tokio's adoption today is its steep learning
curve, an opinion shared by a large number of folks! The number one motivation
of this RFC is to tackle this problem head-on, ensuring that the technical
foundation itself of Tokio is simplified to enable a much smoother introductory
experience into the "world of async" in Rust.

One mistake we made early on in the Tokio project was to so promiently mention
and seemingly recommend the `tokio-proto` and `tokio-service` crates in the
documentation. The `tokio-proto` crate itself is only intended to be used by
authors implementing protocols, which is in theory a pretty small number of
people! Instead though the implementation and workings of `tokio-proto` threw
many newcomers for a spin as they struggled to understand how `tokio-proto`
solved their problem. It's our intention that with this RFC the functionality
provided by the `tokio-proto` and `tokio-service` crates are effectively
moved elsewhere in the ecosystem. In other words, the "Tokio project" as a term
should not invoke thoughts of `tokio-proto` or `tokio-service` as they are
today, but be more solely focused around `tokio-core`.

Anecdotally we've had more success with the `tokio-core` crate being easier to
pick up and not as complicated, but it's not without its own problems. The
distinction between `Core`, `Handle`, and `Remote` is subtle and can be
difficult to grasp, for example. Furthermore we've since clarified that
`tokio-core` is conflating two different concerns in the same crate: spawning
tasks and managing I/O. Our hope is to rearchitect the `tokio-core` crate with a
drastically simpler API surface area to make introductory examples easier to
read and libraries using `tokio-core` easier to write.

It is our intention that after this reorganization happens the introduction to
the Tokio project is a much more direct and smoother path than it is today.
There will be fewer crates to consider (mostly just `tokio-core`) which have a
much smaller API to work with (detailed below) and should allow us to tackle
the heart of async programming, futures, much more quickly in the
documentation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Tokio project, intended to be the foundation of the asynchronous I/O
ecosystem in Rust, is defined by its main crate, `tokio-core`. The `tokio-core`
crate will provide an implementation of an event loop, powered by the
cross-platform `mio` library. The main feature of `tokio-core` is to enable
using I/O objects to implement futures, such as TCP connections, UDP sockets,
etc.

The `tokio-core` crate by default has a global event loop that all I/O will be
processed on. The global event loop enables std-like servers to be created, for
example this would be an echo server written with `tokio-core`:

```rust
extern crate futures;
extern crate tokio;
extern crate tokio_io;

use std::env;
use std::net::SocketAddr;

use futures::{Stream, Future};
use futures::unsync::CurrentThread;
use tokio::net::TcpListener;
use tokio::reactor::Handle;
use tokio_io::AsyncRead;
use tokio_io::io::copy;

fn main() {
    let addr = env::args().nth(1).unwrap_or("127.0.0.1:8080".to_string());
    let addr = addr.parse::<SocketAddr>().unwrap();

    // Notice that unlike today, the `handle` argument is acquired as a global
    // reference rather than from a locally defined `Core`.
    let socket = TcpListener::bind(&addr, Handle::global()).unwrap();
    println!("Listening on: {}", addr);

    let done = socket.incoming().for_each(move |(socket, addr)| {
        let (reader, writer) = socket.split();
        let amt = copy(reader, writer);
        let msg = amt.then(move |result| {
            match result {
                Ok((amt, _, _)) => println!("wrote {} bytes to {}", amt, addr),
                Err(e) => println!("error on {}: {}", addr, e),
            }

            Ok(())
        });

        // Again, unlike today you don't need a `handle` to spawn but can
        // instead spawn futures conveniently through the `CurrentThread` type
        // in the `futures` crate
        CurrentThread.spawn(msg);
        Ok(())
    });

    done.wait().unwrap();
}
```

The purpose of the global event loop is to free users by default from having to
worry about what an event loop is or how to interact with it. Instead most
servers "will just work" as I/O objects, timeouts, etc, all get bound to the
global event loop.

Additionally, unlike today, we won't need to mention `Core` in the documentation
at all. Instead we can recommend beginners to simply use `Handle::global()` to
acquire a reference to a handle, and this architecture may even be the most
appropriate for their use case!

### Spawning in `futures`

The `futures` crate will grow a type named `CurrentThread` which is an
implementation of the `Executor` trait for spawning futures onto the current
thread. This type serves the ability to spawn a future to run "concurrently in
the background" when the thread is otherwise blocked on other futures-related
tasks. For example while calling `wait` all futures will continue to make
progress.

One important change with this is that the ability to "spawn onto a `Core`" is
no longer exposed, and this will need to be constructed manually if desired. For
example usage of `Handle::spawn` today will switch to `CurrentThread.spawn`, and
usage of `Remote::spawn` will need to be manually orchestrated with a
constructed mpsc channel which uses `CurrentThread.spawn` on one end.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Changes to `tokio-core`

This RFC is a backwards-compatible change to `tokio-core` and will be released
as simply a new minor version. The major differences, however will be:

* `Core` is now both `Send` and `Sync`, but methods continue to take `&mut self`
  for `poll` and `turn` to ensure exclusive access when running a `Core`. This
  restriction may also be lifted in the future.
* The `Handle` type is now also both `Send` and `Sync`. This removes the need
  for `Remote`. The `Handle` type also has a `global` method to acquire a
  handle to the global event loop.
* All spawning related functionality is removed in favor of implementations in
  the `futures` crate.

The removal of the distinction between `Handle` and `Remote` is made possible
through removing the ability to spawn. This means that a `Core` itself is
fundamentally `Send`-compatible and with a tweak to the implementation we can
get both `Core` and `Handle` to be both `Send` and `Sync`.

All methods will continue to take `&Handle` but it's not required to create a
`Core` to acquire a `Handle`. Instead the `Handle::global` function can be used
to extract a handle to the global event loop.

The deprecated APIs will be:

* `Remote` and all related APIs are deprecated
* `Handle::spawn` is deprecated and reimplemented through `CurrentThread.spawn`
* `Remote::spawn` is deprecated by sending a future to the reactor and using
  `CurrentThread.spawn`, but it's intended that applications should orchestrate
  this themselves rather than using `Remote::spawn`
* The `Executor` implementations on `Core`, `Handle`, and `Remote` are all
  deprecated.

## Global event loop

It is intended that all application architectures using `tokio-core` today will
continue to be possible with `tokio-core` after this RFC. By default, however,
a lazily initialized global event loop will be executed on a helper thread for
each process. Applications can continue, if necessary, to create and run a
`Core` manually to avoid usage `Handle::global`.


Code that currently looks like this will continue to work:

```rust
let mut core = Core::new().unwrap();
let handle = core.handle();
let listener = TcpListener::bind(&addr, &handle).unwrap();
let server = listener.incoming().for_each(/* ... */);
core.run(server).unwrap();
```

although examples and documentation will instead recommend a pattern that looks
like:

```rust
let handle = Handle::global();
let listener = TcpListener::bind(&addr, handle).unwrap();
let server = listener.incoming().for_each(/* ... */);
server.wait().unwrap();
```

## Spawning Futures

One of the crucial abilities of `Core` today, spawning features, is being
removed! This comes as a result of distilling the features that the `tokio`
crate provides to the bare bones, which is just I/O object registration (e.g.
interaction with `epoll` and friends). Spawning futures is quite common today
though, so we of course still want to support it!

This support will be added through the `futures` crate rather than the
`tokio` crate itself. Namely the `futures` crate effectively already has an
efficient implementation of spawning futures through the `FuturesUnordered`
type. To expose this, the `futures` crate will grow the following type in the
`futures::unsync` module:

```rust
// in futures::unsync

pub struct CurrentThread;

impl CurrentThread {
    /// Spawns a new future to get executed on the current thread.
    ///
    /// This future is added to a thread-local list of futures. This list of
    /// futures will be "completed in the background" when the current thread is
    /// otherwise blocked waiting for futures-related work. For example calls to
    /// `Future::wait` will by default attempt to run futures on this list. In
    /// addition, external runtimes like the `tokio` crate will also execute
    /// this list of futures in the `Core::run` method.
    ///
    /// Note that this can be a dangerous method to call if you don't know what
    /// thread you're being invoked from. The thread local list of futures is
    /// not guaranteed to be moving forward, which could cause the spawned
    /// future here to become inert. It's recommended think carefully when
    /// calling this method and either ensure that you're running on a thread
    /// that's moving the list forward or otherwise document that your API
    /// requires itself to be in such a context.
    ///
    /// Also note that the `future` provided here will be entirely executed on
    /// the current thread. This means that execution of any one future will
    /// block execution of any other future on this list. You'll want to
    /// accordingly ensure that none of the work for the future here involves
    /// blocking the thread for too long!
    pub fn spawn<F>(&self, future: F)
        where F: Future<Item = (), Error = ()> + 'static;

    /// Attempts to complete the thread-local list of futures.
    ///
    /// This API is provided for *runtimes* to try to move the thread-local list
    /// of futures forward. Each call to this function will perform as much work
    /// as possible as it can on the thread-local list of futures.
    ///
    /// The `notify` argument here and `id` are passed with similar semantics to
    /// the `Spawn::poll_future_notify` method.
    ///
    /// In general unless you're implementing a runtime for futures you won't
    /// have to worry about calling this method.
    pub fn poll<T>(&self, notify: &T, id: usize)
        where T: Clone + Into<NotifyHandle>;
}

impl<F> Executor<F> for CurrentThread
    where F: Future<Item = (), Error = ()> + 'static
{
    // ...
}
```

The purpose of this type is to retain the ability to spawn futures referencing
`Rc` and other non-`Send` data in an efficient fashion. The `FuturesUnordered`
primitive used to implement this should exhibit similar performance
characteristics as the current implementation in `tokio-core`.

## Fate of other Tokio crates

This RFC proposed deprecating the `tokio-proto` and `tokio-service` crates
*within the Tokio project*. It's intended that the purpose of these crates will
be taken over by higher level projects rather than continuing to be equated with
the "Tokio" project and moniker. The documentation of the Tokio project will
reflect this by getting updated to exclusively discuss `tokio-core` and the
abstractions that it provides.

The `tokio-service` and `tokio-proto` crates will not be immediately deprecated,
but they will likely be deprecated once a replacement arises within the
ecosystem. The documentation, again, will make far fewer mentions of these
crates relative to `futures` in general and the `tokio-core` crate.

## Migration Guide

As mentioned before, this RFC is a backwards-compatible change to the
`tokio-core` crate. The new deprecations, however, can be migrated via:

* Usage of `Remote` can be switched to usage of `Handle`.
* Usage of `Core` can largely get removed in favor of `Handle::global`.
* Usage of `Handle::spawn` or `Executor for Core` can be replaced with the
  `CurrentThread` type in the `futures` crate.
* Usage of `Remote::spawn` must be rearchitected locally with a manually created
  mpsc channel and `CurrentThread`.

APIs will otherwise continue to take `&Handle` as they do today! Further
non-backwards-compatible fixes to the `tokio-core` crate are deferred for now in
favor of a future RFC.

# Drawbacks
[drawbacks]: #drawbacks

This change is inevitably going to create a fairly large amount of churn with
respect to the `tokio-proto` and `tokio-service` crates, and this will take
some time to propagate throughout the ecosystem.

Despite this, however, we view the churn as worth the benefits we'll reap on the
other end. This change will empower us to greatly focus the documentation of
Tokio solely on the I/O related aspects and free up future library and framework
design space. This should also bring much more clarity to "what is the Tokio
project?" and at what layer you enter at, as there's only one layer!

# Rationale and Alternatives
[alternatives]: #alternatives

As with any API design there's a lot of various alternatives to consider, but
for now this'll be limited to more focused alternatives to the current "general
design" rather than more radical alternatives that may look entirely different.

TODO: more docs here

# Unresolved questions
[unresolved]: #unresolved-questions

N/A

# Summary
[summary]: #summary

Simplify the Tokio project by renaming the `tokio-core` crate to `tokio`,
greatly reducing ergonomic issues in the existing `tokio-core` API, and
de-emphasize `tokio-proto` and `tokio-service` in the online documentation.

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
solved their problem. It's our intention that with this RFC the `tokio-proto`
and `tokio-service` crates are effectively deprecated. This will provide a
strong signal that users and newcomers should be entering at a different point
of the stack.

Anecdotally we've had more success with the `tokio-core` crate being easier to
pick up and not as complicated, but it's not without its own problems. The
distinction between `Core`, `Handle`, and `Remote` is subtle and can be
difficult to grasp, for example. Furthermore we've since clarified that
`tokio-core` is conflating two different concerns in the same crate: spawning
tasks and managing I/O. Our hope is to rearchitect the `tokio-core` crate with a
drastically simpler API surface area to make introductory examples easier to
read and libraries using `tokio-core` easier to write.

Finally one of the main points of confusion around the Tokio project has been
around the many layers that are available to you. Each crate under the Tokio
umbrella provides a slightly different entry point and is intended for different
audiences, but it's often difficult to understand what audience you're supposed
to be in! This in turn has led us to the conclusion that we'd like to rename the
`tokio-core` crate under the new name `tokio`. This crate, `tokio`, will be the
heart soul of the Tokio project, deferring frameworks like the functionality
`tokio-proto` and `tokio-service` provide to different libraries and projects.

It is our intention that after this reorganization happens the introduction to
the Tokio project is a much more direct and smoother path than it is today.
There will be fewer crates to consider (just `tokio`) which have a much smaller
API to work with (detailed below) and should allow us to tackle the heart of
async programming, futures, much more quickly in the documentation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Tokio project, intended to be the foundation of the asynchronous I/O
ecosystem in Rust, is defined by its main crate, `tokio`. The `tokio` crate will
provide an implementation of an event loop, powered by the cross-platform `mio`
library. The main feature of `tokio` is to enable using I/O objects to implement
futures, such as TCP connections, UDP sockets, etc.

The `tokio` crate by default has a global event loop that all I/O will be
processed on. The global event loop enables std-like servers to be created, for
example this would be an echo server written with `tokio`:

```rust
extern crate futures;
extern crate tokio;
extern crate tokio_io;

use std::env;
use std::net::SocketAddr;

use futures::Future;
use futures::stream::Stream;
use futures::unsync::CurrentThread;
use tokio::net::TcpListener;
use tokio_io::AsyncRead;
use tokio_io::io::copy;

fn main() {
    let addr = env::args().nth(1).unwrap_or("127.0.0.1:8080".to_string());
    let addr = addr.parse::<SocketAddr>().unwrap();

    // Notice that unlike today, no `handle` argument is needed when creating a
    // `TcpListener`.
    let socket = TcpListener::bind(&addr).unwrap();
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

### Spawning in `futures`

The `futures` crate will grow a type named `CurrentThread` which is an
implementation of the `Executor` trait for spawning futures onto the current
thread. This type serves the ability to spawn a future to run "concurrently in
the background" when the thread is otherwise blocked on other futures-related
tasks. For example while calling `wait` all futures will continue to make
progress.

One important change with this is that the ability to "spawn onto a `Core`" is
no longer exposed, and this will need to be constructed manually if desired.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The `tokio` crate API

The new `tokio` crate will have a similar API to `tokio-core`, but with a few
notable differences:

* All APIs that take `&Handle` today will have this argument removed. For
  example `Timeout::new(dur)` instead of `Timeout::new(dur, &handle)`.
* The `Remote` and `Handle` types are removed
* `Core` is now both `Send` and `Sync`
* Public methods on `Core` take `&self` instead of `&mut self`.

The removal of the `&Handle` arguments are made possible through the event loop
management detailed below. The removal of `Handle` and `Remote` is made possible
through two separate avenues:

* First and foremost, the ability to `spawn` futures onto a `Core` has been
  removed. This is discussed more in the spawning futures section below.
* Next, the `Core` type itself is now `Send` and `Sync`, namely allowing for
  concurrent registration of I/O objects and timeouts.

The remaining APIs of `Core` are the `run` and `turn` methods. Although `Core`
is `Send` and `Sync` it's not intended to rationalize the behavior of concurrent
usage of `Core::run` just yet. Instead both of these methods are explicitly
documented as "this doesn't make sense to call concurrently". It's expected that
in the future we'll rationalize a story for concurrently calling these methods,
but that's left to a future RFC.

## Event loop management

It is intended that all application architectures using `tokio-core` today will
continue to be possible with `tokio` tomorrow. By default, however, a lazily
initialized global event loop will be executed on a helper thread for each
process. Applications can continue, if necessary, to create and run a `Core`
manually to avoid usage of the helper thread and its `Core`.

All threads will have a "default `Core`" associated with them. When a thread
starts this default `Core` is the global `Core`. There will be a few APIs within
the `reactor` module to modify this, however:

```rust
impl Core {
    /// Acquire a reference to this thread's current core.
    ///
    /// This function may return different values when called over time, but the
    /// core returned here is the core that all I/O objects and timeouts will be
    /// associated with on construction.
    ///
    /// # Errors
    ///
    /// If this function falls back to creating the global `Core` it may return
    /// an error, which is indicated through the return value.
    pub fn current() -> io::Result<Core>;

    /// Acquire a reference to the global default `Core`.
    ///
    /// This function will acquire a reference to the global `Core` which is
    /// being executed on a helper thread in the application. The core returned
    /// here is the same each time this function is called.
    ///
    /// # Errors
    ///
    /// If this function initializes the global `Core` it may return
    /// an error, which is indicated through the return value.
    pub fn global() -> io::Result<Core>;

    /// Flags this core as the "current core" for all future operations.
    ///
    /// Calling this function affects the return value of `Core::current`. The
    /// most recent call to `push_current` in this thread is what will be
    /// returned from `Core::current`. When the `PushCurrent` struct here falls
    /// out of scope then the previous current core will be restored.
    pub fn push_current<'a>(&'a self) -> PushCurrent<'a>;
}
```

The purpose of these APIs is to:

* Allow the ability to customize what `Core` I/O objects are registered with.
  Custom-created cores can use `push_current` to ensure that they're used for
  I/O and timeout registration.
* If desired, I/O objects and timeouts can be explicitly registered with the
  global core instead of the currently listed core.

The current core will be managed through a TLS variable, and the global core
will be managed internally by the `tokio` crate.

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

This RFC proposed deprecating the `tokio-core`, `tokio-proto`, and
`tokio-service` crates. The `tokio-proto` and `tokio-service` crates will be
deprecated without replacement for now, but it's expected that higher level
application frameworks will quickly fill in the gap here. For example most users
already aren't using `tokio-proto` (it's just an implementation detail of
`hyper`). Crates like `hyper`'s exposure of `tokio-service` will be refactored
in an application-specific manner, likely involving a Hyper-specific trait.

The `tokio-core` crate, however, will be deprecated with the `tokio` crate as a
replacement. The migration path should be relatively straightforward,
essentially deleting all `Core`, `Remote`, and `Handle` usage while taking
advantage of `CurrentThread`. The `tokio-core` crate will be *reimplemented*,
however, in terms of the new `tokio` crate. A new API will be added to extract a
`tokio::reactor::Core` from a `tokio_core::reactor::Handle`. That is, you can
acquire non-deprecated structures from deprecated structures.

## API migration guidelines

Crates with `tokio-core` as a public dependency will be able to upgrade to
`tokio` without an API breaking change. The migration path looks like:

* If `TcpStream` is exposed in a public API, the API should be generalizable to
  an instance of `AsyncRead + AsyncWrite`. The traits here should in theory be
  enough to not require a `TcpStream` specifically.
* Similarly if a `UdpSocket` is exposed it's intended that `Sink` and `Stream`
  can likely be used instead.
* If `Handle` is exposed then backwards compatibility can be retained by:
  * Deprecate the method taking a `Handle`
  * Add a new method which *doesn't* take a handle with a "worse name". For
    example `UnixListener::bind` would become `UnixListener::bind2`.
  * The previous method can be implemented in terms of the new method via the
    ability to acquire a `Core` from a `Handle` and then usage of
    `push_current`.

Our hope is that crates which have a major version bump themselves on the
horizon can take advantage of the opportunity to "switch" the apis here to have
the "good name" not take a `Handle` and `tokio-core` support can be dropped.

An example of the migration path for `Handle` looks like:

* Today there's a `UnixListener::bind(&Path, &Handle)` API
* After `tokio` is published a `UnixListener::bind2(&Path)` API will be added
* The `UnixListener::bind` API is deprecated
* The `UnixListener::bind` API is implemented by calling `bind2`
* When the `UnixListener` crate has a major version bump the `bind2` API is
  removed, the `bind` function removes the `Handle` argument, and the
  `tokio-core` dependency is dropped.

# Drawbacks
[drawbacks]: #drawbacks

This change is inevitably going to create a fairly large amount of churn. The
migration from `tokio-core` to `tokio` will take time, and furthermore the
deprecation of `tokio-proto` and `tokio-service` will take some time to
propagate throughout the ecosystem.

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

### TLS `Core` vs handles

Right now the "current core" is managed through thread-local state via the
`current`, `global`, and `push_current` APIs. An alternative, however, is to
have two versions of all APIs: one that doesn't take a handle and one that does.
For example:

```rust
impl TcpStream {
    pub fn connect(addr: &SocketAddr) -> TcpStreamNew;
    pub fn connect_core(addr: &SocketAddr, core: &Core) -> TcpStreamNew;
}
```

This would allow us to remove the `current`, `global`, and `push_default` APIs
on `Core` as you could then explicitly configure what `Core` an I/O object is
registered with via `*_core` methods. The benefit of this approach is that you
can no longer accidentally spawn work onto the default core instead of the
global core. The thinking is that the non-`Handle` methods are always "this is
global" and the `*_core` methods are "this is a specific core".

The downside of this approach, however, is that the API surface area duplication
is required in all consumers of the `tokio` crate. For example not just
`TcpStream` would be required to have two methods but anything internally
creating a `TcpStream` would require two methods. Furthermore the downside of
this approach is that you may spawn work onto the global core when you intended
the local core. If a library you're using *doesn't* have the duplicate API
surface area yet, you're forced to use it on the global core if you'd otherwise
have a local core.

# Unresolved questions
[unresolved]: #unresolved-questions

N/A

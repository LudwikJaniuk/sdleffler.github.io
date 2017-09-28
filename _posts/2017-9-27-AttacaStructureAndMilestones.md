---
layout: post
title: The architecture of Attaca, milestones, and current progress
published: false
---

In this post I want to talk a little about my current project,
[Attaca](https://github.com/sdleffler/attaca).

Attaca is a distributed version control system designed for users working with
absurdly large quantities of data. For example, scientists: a brain scan will
routinely result in a 50GB file. Or, video editors, who work with absurdly
large quantities of data in the form of video and image data. Whatever your use
case, Attaca is designed to efficiently handle repositories of up to a petabyte
of data in a distributed storage cluster.

## How?

Attaca is based around three main concepts/technologies: hashsplitting, the Git
data structure, and distributed storage. These combine to let us handle
ridiculously large amounts of data by first, efficiently handling updates to
extremely large files; second, efficiently storing new versions of the
repository; and three, holding our data in a failure-resistant manner. Attaca
is currently built on the Ceph RADOS (Reliable Autonomous Distributed Object
Store) but other backends are planned, as the only required feature of any
backing store is the ability to act as a distributed hash table for fairly
large values (about 3MB per object). It does make a few modifications to the
Git data structure; a detailed explanation of hashsplitting and Attaca's take
on the Git data structure can be found [here]({{ site.baseurl }}{% post_url
2017-8-28-Hashsplitting %}), in one of my earlier blog posts.

## Milestones

When I started working on this project, three milestones were set, estimated at about a month's worth of time for each:

1. Create a client capable of hashsplitting files and storing the resulting chunks in the remote object store.
2. Implement subtrees, "large" blobs, and and commits.
3. Implement the ref store interface, and add local index/commit/push functionality.

Three months in, I can comfortably say I've completed my first milestone!
Hashsplitting and "marshalling" (the process of constructing large blobs from
small blobs) work, and the attaca client is capable of spitting the completed
blobs into a Ceph cluster.

What exactly did this entail, and why did it take so long? There were several
reasons: a lack of idiomatic Rust bindings for Ceph, my unfamiliarity with
hashsplitting, rolling hashes, and hashes in general, and my unfamiliarity with
the `futures` library. Additionally, there was a design problem I didn't expect
when starting the project, and for which I still lack a perfect solution.

## Subproject: `rad-rs`, high-level bindings for librados

When I first looked into using Ceph for this project, I thought, "great! A
system which is much easier to deploy than other distributed object stores and
suits my use case perfectly", which in general is true. However, the Ceph
bindings for Rust covered only the C library, and went no further in terms of
idiomatic Rust. For the first roughly four weeks of this project, I worked on
fixing that problem. The end result was the
[rad](https://github.com/sdleffler/rad-rs) crate, a set of high-level,
idiomatic Rust bindings for librados, the Ceph RADOS client API.

### Asynchronous I/O with Ceph

Ceph and librados offer the ability to do asynchronous reads, writes, and other
operations, which is vital for doing fast network I/O, especially for the
amount of data attaca has to manage. Ceph's C API does this through
"completions", which are C structs constructed through calls to librados API
functions:

```c
rados_completion_t comp;
// These NULL values are, in order:
// - a pointer to custom data to pass to callbacks
// - a pointer to a callback to execute when the operation is "complete"
// - a pointer to a callback to execute when the operation is "safe"
err = rados_aio_create_completion(NULL, NULL, NULL, &comp);
if (err < 0) {
    // Handle error
}
```

Now that you have a completion, you can schedule asynchronous I/O to happen with the given completion, and then wait for the I/O operation to complete:

```c
// Asynchronously write "bar" to object "foo".
err = rados_aio_write(io, "foo", comp, "bar", 3, 0);
if (err < 0) {
    // Handle error
}
rados_aio_wait_for_complete(comp); // in memory
rados_aio_wait_for_safe(comp); // on disk
```

Examples here are taken from the [Ceph librados
docs](https://docs.ceph.com/docs/master/rados/api/librados/).

There are a couple of snags here; some are about idiomatic Rust, and the last
is about the librados API. Let's start with the Rust snags.

#### Completions? No, futures

First off, it doesn't work very well to simply Rust-ify the C API; completions
aren't very... Rusty. First of all, everything's raw pointers, but that can be
easily fixed. Second of all, completions aren't very composable, which is
somewhat against the Rust creed of zero-cost abstractions: iterators, `Option`
and `Result` are all nice and composable. Rust *does* have an answer to the
issue of composable asynchronous computation, and that answer is the
[futures](https://github.com/alexcrichton/futures-rs) crate. What I did in rad
was implement a safe, futures based wrapper around completions. How does this
work?

It's not actually that complicated. Here's the `Completion` type, taken straight from rad:

```rust
#[derive(Debug)]
pub struct Completion<T> {
    task: Arc<AtomicTask>,
    data: Option<Arc<T>>,
    handle: rados_completion_t,
}
```

A few things to note: we have the `rados_completion_t` type from the
[ceph-rust](https://github.com/ceph/ceph-rust) library, which I used for its
raw C bindings. Then, we have the `AtomicTask` type from the futures crate. Why
do we need this?

The "futures model" - on which an in-depth discussion can be found
[here](https://tokio.rs/docs/going-deeper-futures/futures-model/) - is based
around the idea of polling and notifying asynchronous "tasks". When a task is
done, it notifies all its potential consumers, so that they can go grab the
value and keep computing; when a consumer polls a task, it registers itself as
interested in the currently computing value, or if the task is done, it grabs
the computed value and continues on its merry way. `AtomicTask` provides two
basic building blocks of this framework: `AtomicTask::notify` and
`AtomicTask::register`. When a consumer comes along and fines that a
`Completion`'s value is not yet ready, it `.register()`s itself, and then
returns `Async::NotReady`, which tells the environment that the future won't be
waking up for a while. Then, we register a callback on the `rados_completion_t`
type, which runs `.notify()` on the `AtomicTask`, telling all the consumers
that the value is ready.

There are a couple more tricks to this: first of all, the use of `Arc<T>` to
hold arbitrary data for the completion is for two reasons: an `Arc<T>` is used
instead of a `Box` so that we can make our data "available" by creating exactly
two references to the `Arc`'d data and destroying one, thus making it possible
to `Arc::try_unwrap` the data. Second, we need to keep our data's location in
memory stable: the sort of thing we'd put in our completion's `data` is a
buffer for an asynchronous read operation to write its result to. We use an
`Arc<AtomicTask>` for a less interesting reason: `AtomicTask` is `Send` and
`Sync` but not `Clone`. If we're going to share it between the data we put in
our callbacks and the `Completion` type so that our consumers can `.register()`
themselves on it, it has to be reference-counted.

For more information, please refer to the commented code
[here](https://github.com/sdleffler/rad-rs/blob/master/src/async.rs); I think
I've done a pretty good job at keeping it readable.

### C FFI requires `CString`s, not `&str`s

The librados C API uses null-terminated strings for object names in RADOS; this
is at odds with Rust's `&str` string slices. With rad, I settled on the
solution of using a global, thread-safe pool of pre-allocated `CString`
objects. I built a small crate for doing this; it is available on crates.io as
well as on github [here](https://github.com/sdleffler/ffi-pool-rs). Combined
with the incredible
[error-chain](https://github.com/rust-lang-nursery/error-chain) library, this
makes dealing with C strings easy with minimal allocations. Errors
corresponding to failure to convert Rust strings into C strings are translated
into errors in the `rad::Error` type, making error handling homogeneous over
the whole crate.

### Sometimes, futures need to be `'static` and `Send`

The `rados_aio_read` call turns out to be a bit tricky to create a future for.
Since `rados_aio_read` asks for a pointer to a byte buffer and a buffer size,
the obvious thing to do is to pass in an `&mut [u8]`, which we turn into a
`*mut libc::c_char` to pass to RADOS and say, "hey, here's our buffer." This
has problems, however, when we try to do things like use `futures_cpupool` to
run our futures. What's the type signature of `CpuPool::spawn`?

```rust
impl CpuPool {
    pub fn spawn<F>(&self, f: F) -> CpuFuture<F::Item, F::Error>
        where F: Future + Send + 'static,
              F::Item: Send + 'static,
              F::Error: Send + 'static,
    {
        ...
    }
}
```

That's right - our future *has* to be `'static` and `Send`. `Send` makes sense;
running our future in a threadpool is a pretty good way to make sure the future
wakes up in a different thread. `'static`, however, is a little harder to
decipher; "but what about scoped threads!?" you might cry? Well, there are many
possible reasons for it; a few of my guesses are:

- `thread::spawn` requires that the closure it runs is `'static`, and
  `futures_cpupool` didn't want to implement a scoped thread mechanism or draw
  in the dependency of a scoped thread library.

- `CpuPool` being atomically reference-counted makes everything easier, adding
  a lifetime to `CpuPool` *might* make everyones' lives drastically harder?

- The futures model itself is essentially an event loop, and `CpuPool` has to
  implement that; if a `CpuFuture` goes out of scope, the corresponding future
  which was consumed to spawn said `CpuFuture` might not be destroyed until a
  significant time later, at which point it might have outlived its lifetime,
  making dropping it potentially unsafe and capable of undefined behavior.

Personally, I believe the last to be the most likely possibility. In any case
the ramifications of this are that while it's okay for futures running outside
of a thread pool to be non-`'static`, I need any future I want to run in a
separate thread to be `'static`; if my `ReadFuture` has a significant lifetime
bound, everything falls apart. My solution was to pull out the
[stable_deref_trait](https://github.com/storyyeller/stable_deref_trait)
library, and allow the buffer passed in to be *any* type which can stably
dereference to an `&mut [u8]`; this allows `Vec<u8>` to be used interchangeably
with fixed-size buffers such as `[u8; 4096]` and even mutable references to
byte slices (`&'a mut [u8]`), the last of which introduces a lifetime to the
`ReadFuture<B>` type. Doing this allows buffers passed in to have a lifetime if
necessary, thus giving the future a non-`'static` lifetime, *or* be `'static`,
in which case the future will also be `'static`.  In either case the buffer is
returned as part of the future's result value: when a read is complete, you get
ownership of your buffer back, whether it was borrowed in the first place or
temporarily owned by the future.

### A small but significant piece of inaccurate documentation

After three months of work, I was hit with a particularly interesting surprise:
there is an odd (and generally harmless) inaccuracy in the Ceph/librados docs.
While `rados_completion_t`s in the C API have both "complete" and "safe"
callbacks, indicating when a completion's operation is completely in memory in
the RADOS cluster and completely written to disk (respectively), this
distinction has actually been removed from the underlying code. As such, both
the "complete" and "safe" callbacks are called when the operation is finished
and on-disk on the cluster - the "complete" callback in the latest versions of
Ceph is identical in function to the "safe" callback (although the "complete"
callback is always run first.) I've used this distinction to my advantage in my
rad crate - however, this does mean that rad will behave oddly when used with
versions of Ceph released before Kraken (Luminous is the most recent release;
Ceph releases run in alphabetical order (*H*ammer, *I*nfernalis, *J*ewel,
*K*raken, *L*uminous.) There is an open issue to fix rad so that it checks the
librados version upon creating a cluster handle, and panics in case of a
version of Ceph that is too old.

## Hashsplitting: don't invent your own

I spent a couple of weeks working on hashsplitting. All in all, hashsplitting
is very simple: just use the
sum-all-bytes-in-a-window-and-check-if-it's-divisible-by-your-power-of-two-of-choice
method. That's what rsync and bup do, and it works great.

I went down a bit of a wild goose chase: I attempted, at first, to tune the
adler32 rolling hash to my purposes. I attempted to use an overly clever "count
the bits that match and see if the number of bits that match hit a threshold"
condition. None of this worked - the overall distribution of chunk sizes that
resulted was... discouraging. You'd get a smattering of too-small chunks and
then several *very* large chunks.

If you're going to hashsplit, adler32 might work for you - in fact, I suspect
that if I went back and used it, it'd work fine (the main problem with it was
the overly clever "if then chunk" condition.) However, simply summing bytes
also works just fine, although it has a couple of edge cases. Don't overthink
it, and don't waste too much time on it - it's a lot simpler than it might
sound at first glance.

## Futures: how to make your program actually run concurrently

When I first
[implemented](https://github.com/sdleffler/attaca/commit/53ffaa2088c1562f1a4ed9ed729153116712a7b1)
asynchronous basically-everything-that-mattered for attaca, I ran into a
problem.

It wasn't actually running concurrently. At all.

And it took me a fairly significant amount of time to figure out exactly why.
Here's a summary of my newbie mistakes and various lessons learned with the
futures crate. If you're familiar with futures and you've used it for
concurrent programming before, you might as well skip this bit - or maybe read
it and tell me what more I'm doing wrong, because I'd love to hear advice from
someone more familiar with the library.

### Don't forget to buffer

My first mistake was not using the `Stream::buffered` and
`Stream::buffer_unordered` methods. *Nothing will actually run concurrently on
top of a `Stream` unless you buffer futures or take them out before you run
them; using `Stream::and_then` will block the top of the `Stream`.* In my case,
I was constructing a stream, using `Stream::and_then` to do some computation on
top, and then using `Stream::for_each` to drive the stream to completion.

The end result of this was that `for_each` would wait for the future at the top
of the stream to complete. Then, it would poll the stream for the next future
(which would cause the future to be constructed *and then* start executing, and
wait for this one to complete. Et cetera ad infinitum.

You can see why this might be slow. We have no concurrency! We're starting
futures and waiting for them to complete one at a time. And this is where
`Stream::buffer_unordered` and `Stream::buffered` come to the rescue: if you're
constructing a bunch of futures off the top of a `Stream`, and you want them to
be run concurrently, *don't* use `Stream::and_then`; use `Stream::map` to
create a stream of futures, and then use `Stream::buffered` or
`Stream::buffer_unordered`. This avoids waiting for each individual future to
complete and instead allows you to run `N` futures concurrently.

### Blocking is not always bad

If your program is running concurrently with futures, note that whenever a
future isn't ready, it gives another future a chance to run. In my program, I
saw no concurrency until I lowered buffer sizes: producer tasks would produce
until they ran out of items to produce, and never gave the consumer task a
chance to consume anything. Reducing the buffer sizes meant that when the
producers hit the buffer limit, they become blocked, causing the futures
library to give execution to an unblocked task - the consumer! I'm still
looking for a way to cause consumption without the producer hitting a buffer
limit, but at the moment, this is the only reason anything works concurrently
in attaca.

### `futures_await` is fantastic

Writing with futures and using `Futures::and_then` all the time can get very
verbose and difficult to read. I've started to take apart bits of my code and
replace them with the `async_block!` macro from the
[`futures-await`](https://github.com/alexcrichton/futures-await) crate.
Funnily enough, some of the futures code I wrote before was so gnarly that
`rustfmt` was having trouble formatting it - but using the `await!` and
`async_block!` macros, it came so close to regular Rust with `Result`-based
error handling that it's not only readable but also formats nicely (although I
have to remove the `async_block!` macro each time I run `rustfmt`, since
`rustfmt` won't format macro calls.)

Using `futures_await` will put you on nightly, but it's worth it.

## It's hard to make a tree when you don't have all the data

Building a "large blob" means making a tree out of "small" blobs. This is part
of how attaca efficiently stores large files: a very large file is a tree
structure, where "large" blobs are branch nodes and "small" blobs are leaf
nodes, each holding a roughly 3MB chunk of the original file.

Hidden in this was a major concern for me: eventually, it should be possible to
`attaca checkout` (disclaimer: as of now, command only exists in concept and
not in reality) a *slice* of a file instead of cloning the entire file to disk.
The reason for this would be to lazily load bits of a file through something
like SAMBA or FUSE, allowing for users without terabytes upon terabytes of
storage in their machines to work with a repository containing petabytes of
data. Concretely, this means that if the user makes changes and commits them
without the entirety of the multi-blob file in-memory on their local machine,
then we have to load blobs from a remote store *while* constructing the
multi-blob file tree structure - which is at odds with one of my design goals,
which was to keep the process of marshalling blobs free of queries to remotes.
Eventually, I decided to put aside the exact design of how to do this for a
later date; currently, I assume that all large blobs for a given file are in
memory and that we have size information for all small blobs of consequence.
This is enough to construct and push any large blob we want to a remote
repository.

## Conclusions to draw and current progress

As of now, most of the internal machinery is in place to implement subtrees,
commits, and build a git-like index structure! Now that most of the
implementation details are out of the way (fingers crossed...) it should be a
bit easier to implement new structures and operations on remote and local
object stores. I'll be pushing towards this goal over the next few weeks.

# Futures in Rust

> **Overview:**
>
> - Get a high level introduction to concurrency in Rust
> - Know what Rust provides and not when working with async code
> - Get to know why we need a runtime-library in Rust
> - Understand the difference between  "leaf-future" and a "non-leaf-future"
> - Get insight on how to handle CPU intensive tasks

## Futures

So what is a future?

A future is a representation of some operation which will complete in the
future.

Async in Rust uses a `Poll` based approach, in which an asynchronous task will
have three phases.

1. **The Poll phase.** A Future is polled which results in the task progressing until
a point where it can no longer make progress. We often refer to the part of the
runtime which polls a Future as an executor.
2. **The Wait phase.** An event source, most often referred to as a reactor,
registers that a Future is waiting for an event to happen and makes sure that it
will wake the Future when that event is ready.
3. **The Wake phase.** The event happens and the Future is woken up. It's now up
to the executor which polled the Future in step 1 to schedule the future to be
polled again and make further progress until it completes or reaches a new point
where it can't make further progress and the cycle repeats.

Now, when we talk about futures I find it useful to make a distinction between
**non-leaf** futures and **leaf** futures early on because in practice they're
pretty different from one another.

### Leaf futures

Runtimes create _leaf futures_ which represent a resource like a socket.

```rust, ignore, noplaypen
// stream is a **leaf-future**
let mut stream = tokio::net::TcpStream::connect("127.0.0.1:3000");
```

Operations on these resources, like a `Read` on a socket, will be non-blocking
and return a future which we call a leaf future since it's the future which
we're actually waiting on.

It's unlikely that you'll implement a leaf future yourself unless you're writing
a runtime, but we'll go through how they're constructed in this book as well.

It's also unlikely that you'll pass a leaf-future to a runtime and run it to
completion alone as you'll understand by reading the next paragraph.

### Non-leaf-futures

Non-leaf-futures are the kind of futures we as _users_ of a runtime write
ourselves using the `async` keyword to create a **task** which can be run on the
executor.

The bulk of an async program will consist of non-leaf-futures, which are a kind
of pause-able computation. This is an important distinction since these futures represents a _set of operations_. Often, such a task will `await` a leaf future
as one of many operations to complete the task.

```rust, ignore, noplaypen, edition2018
// Non-leaf-future
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap();// <- yield
    println!("connected!");
    let result = stream.write(b"hello world\n").await; // <- yield
    println!("message sent!");
    ...
};
```

The key to these tasks is that they're able to yield control to the runtime's
scheduler and then resume execution again where it left off at a later point.

In contrast to leaf futures, these kind of futures do not themselves represent
an I/O resource. When we poll them they will run until they get to a
leaf-future which returns `Pending` and then yield control to the scheduler
(which is a part of what we call the runtime).

## Runtimes

Languages like C#, JavaScript, Java, GO, and many others comes with a runtime
for handling concurrency. So if you come from one of those languages this will
seem a bit strange to you.

Rust is different from these languages in the sense that Rust doesn't come with
a runtime for handling concurrency, so you need to use a library which provides
this for you.

Quite a bit of complexity attributed to Futures is actually complexity rooted
in runtimes; creating an efficient runtime is hard.

Learning how to use one correctly requires quite a bit of effort as well, but
you'll see that there are several similarities between these kind of runtimes, so
learning one makes learning the next much easier.

The difference between Rust and other languages is that you have to make an
active choice when it comes to picking a runtime. Most often in other languages,
you'll just use the one provided for you.

### A useful mental model of an async runtime

I find it easier to reason about how Futures work by creating a high level mental model we can use.
To do that I have to introduce the concept of a runtime which will drive our Futures to completion.

>Please note that the mental model I create here is not the **only** way to drive Futures to
completion and that Rust’s Futures does not impose any restrictions on how you actually accomplish
this task.

**A fully working async system in Rust can be divided into three parts:**

1. Reactor
2. Executor
3. Future

So, how does these three parts work together? They do that through an object called the `Waker`.
The `Waker` is how the reactor tells the executor that a specific Future is ready to run. Once you
understand the life cycle and ownership of a Waker, you'll understand how futures work from a user's
perspective. Here is the life cycle:

- A Waker is created by the **executor.**
- When a future is polled the first time by the executor, it’s given a clone of the Waker
  object created by the executor. Since this is a shared object (e.g. an
  `Arc<T>`), all clones actually point to the same underlying object.  Thus,
  anything that calls _any_ clone of the original Waker will wake the particular
  Future that was registered to it.
- The future clones the Waker and passes it to the reactor, which stores it to
  use later.
  
You could think of a "future" like a channel for the `Waker`: The channel starts with the future that's polled the first time by the executor and is passed a handle to a `Waker`. It ends in a leaf-future which passes that handle to the reactor.

>Note that the `Waker` is wrapped in a rather uninteresting `Context` struct which we will learn more about later. The interesting part is the `Waker` that is passed on.

At some point in the future, the reactor will decide that the future is ready to run. It will wake the future via the Waker that it stored. This action will do what is necessary to get the executor in a position to poll the future. We'll go into more detail on Wakers in the [Waker and Context chapter.](3_waker_context.md#understanding-the-waker)

Since the interface is the same across all executors, reactors can _in theory_ be completely
oblivious to the type of the executor, and vice-versa. **Executors and reactors never need to
communicate with one another directly.**

This design is what gives the futures framework it's power and flexibility and allows the Rust
standard library to provide an ergonomic, zero-cost abstraction for us to use.

In an effort to try to visualize how these parts work together I put together
a set of slides in the next chapter that I hope will help.

The two most popular runtimes for Futures as of writing this is:

- [async-std](https://github.com/async-rs/async-std)
- [Tokio](https://github.com/tokio-rs/tokio)

### What Rust's standard library takes care of

1. A common interface representing an operation which will be completed in the
future through the `Future` trait.
2. An ergonomic way of creating tasks which can be suspended and resumed through
the `async` and `await` keywords.
3. A defined interface to wake up a suspended task through the `Waker` type.

That's really what Rust's standard library does. As you see there is no definition
of non-blocking I/O, how these tasks are created, or how they're run.

## I/O vs CPU intensive tasks

As you know now, what you normally write are called non-leaf futures. Let's
take a look at this async block using pseudo-rust as example:

```rust, ignore
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap(); // <-- yield

    // request a large dataset
    let result = stream.write(get_dataset_request).await.unwrap(); // <-- yield

    // wait for the dataset
    let mut response = vec![];
    stream.read(&mut response).await.unwrap(); // <-- yield

    // do some CPU-intensive analysis on the dataset
    let report = analyzer::analyze_data(response).unwrap();

    // send the results back
    stream.write(report).await.unwrap(); // <-- yield
};
```

Now, as you'll see when we go through how Futures work, the code we write between
the yield points are run on the same thread as our executor.

That means that while our `analyzer` is working on the dataset, the executor
is busy doing calculations instead of handling new requests.

Fortunately there are a few ways to handle this, and it's not difficult, but it's
something you must be aware of:

1. We could create a new leaf future which sends our task to another thread and
resolves when the task is finished. We could `await` this leaf-future like any
other future.

2. The runtime could have some kind of supervisor that monitors how much time
different tasks take, and move the executor itself to a different thread so it can
continue to run even though our `analyzer` task is blocking the original executor thread.

3. You can create a reactor yourself which is compatible with the runtime which
does the analysis any way you see fit, and returns a Future which can be awaited.

Now, #1 is the usual way of handling this, but some executors implement #2 as well.
The problem with #2 is that if you switch runtime you need to make sure that it
supports this kind of supervision as well or else you will end up blocking the
executor.

And #3 is more of theoretical importance, normally you'd be happy by sending the task
to the thread-pool most runtimes provide.

Most executors have a way to accomplish #1 using methods like `spawn_blocking`.

These methods send the task to a thread-pool created by the runtime where you
can either perform CPU-intensive tasks or "blocking" tasks which are not supported
by the runtime.

Now, armed with this knowledge you are already on a good way for understanding
Futures, but we're not gonna stop yet, there are lots of details to cover.

Take a break or a cup of coffee and get ready as we go for a deep dive in the next chapters.

## Want to learn more about concurrency and async?

If you find the concepts of concurrency and async programming confusing in
general, I know where you're coming from and I have written some resources to
try to give a high-level overview that will make it easier to learn Rust's
Futures afterwards:

- [Async Basics - The difference between concurrency and parallelism](https://cfsamson.github.io/book-exploring-async-basics/1_concurrent_vs_parallel.html)
- [Async Basics - Async history](https://cfsamson.github.io/book-exploring-async-basics/2_async_history.html)
- [Async Basics - Strategies for handling I/O](https://cfsamson.github.io/book-exploring-async-basics/5_strategies_for_handling_io.html)
- [Async Basics - Epoll, Kqueue and IOCP](https://cfsamson.github.io/book-exploring-async-basics/6_epoll_kqueue_iocp.html)

Learning these concepts by studying futures is making it much harder than
it needs to be, so go on and read these chapters if you feel a bit unsure.

I'll be right here when you're back.

However, if you feel that you have the basics covered, then let's get moving!

## Bonus section - additional notes on Futures and Wakers

> In this section we take a deeper look at some  advantages of having a loose
coupling between the Executor-part and Reactor-part of an async runtime.

Earlier in this chapter, I mentioned that it is common for the
executor to create a new Waker for each Future that is registered with the
executor, but that the Waker is a shared object similar to a `Arc<T>`. One of
the reasons for this design is that it allows different Reactors the
ability to Wake a Future.

As an example of how this can be used, consider how you could create a new type
of Future that has the ability to be canceled:

One way to achieve this would be to add an
[`AtomicBool`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html)
to the instance of the future, and an extra method called `cancel()`.  The
`cancel()` method  will first set the
[`AtomicBool`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html)
to signal that the future is now canceled, and then immediately call instance's
own  copy of the Waker.

Once the executor starts executing the Future, the
_Future_ will know that it was canceled, and will do the appropriate cleanup
actions to terminate itself.

The main reason for designing the Future in this manner is because we don't have
to modify either the Executor or the other Reactors; they are all oblivious to
the change.

The only possible issue is with the design of the Future itself; a
Future that is canceled still needs to terminate correctly according to the
rules outlined in the docs for
[`Future`](https://doc.rust-lang.org/std/future/trait.Future.html).  That means
that it can't just delete it's resources and then sit there; it needs to return
a value.  It is up to you to decide if a canceled future will return
[`Pending`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Pending)
forever, or if it will return a value in
[`Ready`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Ready). Just
be aware that if other Futures are `await`ing it, they won't be able to start
until [`Ready`](https://doc.rust-lang.org/std/task/enum.Poll.html#variant.Ready)
is returned.

A common technique for cancelable Futures is to have them return a
Result with an error that signals the Future was canceled; that will permit any
Futures that are awaiting the canceled Future a chance to progress, with the
knowledge that the Future they depended on was canceled.  There are additional
concerns as well, but beyond the scope of this book.  Read the documentation and
code for the [`futures`](https://crates.io/crates/futures) crate for a better
understanding of what the concerns are.

>_Thanks to [@ckaran](https://github.com/ckaran) for contributing this bonus segment._

[async_std]: https://github.com/async-rs/async-std
[tokio]: https://github.com/tokio-rs/tokio
[compat_info]: https://rust-lang.github.io/futures-rs/blog/2019/04/18/compatibility-layer.html
[futures_rs]: https://github.com/rust-lang/futures-rs

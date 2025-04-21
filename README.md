# Weston's "What Fresh Hell is Rust's Pin...for Dummies (e.g. Weston)"

Congratulations.  You've run into Rust's std::pin::Pin and you have no idea what the hell that means.
This tutorial gives a high level "user overview" of pinning.  If you want a detailed overview, including
why it is needed, see https://doc.rust-lang.org/std/pin/index.html

This overview is not complete.  It verges on patronizing.  If it is any comfort, just know that I wrote
this mostly for myself and so if it feels like I am explaining things to a child that's just a reflection
of my own mental capabilities.

## How did I get here? / Why do I care?

You're probably implementing, or trying to invoke, a method that has self defined as `Pin<&mut Self>`.  This method
is probably `std::future::Future::poll` or `futures::stream::Stream::poll_next`.  If you're trying to invoke the
method, you're probably wondering how you get an instance of Pin<&mut Self>.  If you're trying to implement the
method, you're probably wondering what you can do with it.

If this is not how you got here then maybe you're trying to do self-referential things.  If that's the case then
this guide is not for you.  Maybe try not to do that?  You've gone beyond my limited capabilities.

### Getting an instance of Pin<&mut T> so you can poll a future

To get an instance of `Pin<&mut T>` you normally want to get an instance of `Pin<Box<T>>` first.

If you have an instance of Pin<Box<T>> then you can just call `.as_mut()`.

If you have an instance of T then you can call `Box::pin(thing)` to get an instance of `Pin<Box<T>>`.  This is
saying "I'm done building this future and ready to start polling it".

If you have an instance of `Box<T>` then you can call `Box::pin(*thing)`.

If all you have is `Arc<T>` then you're SOL.  Futures aren't meant to be shared.

Technically, you can convert an instance of T into an `Pin<&mut T>` without consuming `T` by using the `pin!`
macro.  This is weird and probably not what you want unless you really hate heap allocations.

### You're given an instance of Pin<&mut T>, now what?

If you have an instance of Pin<&mut T> then it is somewhere between an `Arc<T>` and a `Box<T>`.  You are
allowed to easily get a shared reference (e.g. `&T`) like an `Arc<T>`.  However, you are not able to easily
get a mutable reference (e.g. `&mut T`) but (unlike an `Arc<T>`) it is kind of possible and you are sort
of meant to do so.

In order to get a mutable reference you have to invoke `unsafe` and promise you won't do bad things.  You
probably don't want to do this because then you need to read the above link to understand what "bad things"
means in this context.

### I'm implementing std::future::Future or futures::stream::Stream and I ran into this problem

Great.  This probably covers 90% of the situations I've personally run into.  You are not alone.

Guess what.  Are you building your own low level library on top of asynchronous kernel routines?  Are you
trying to wrap some other language's async library in a Rust ansyc FFI wrapper?  Are you really implementing
std::future::Future and not just wrapping some existing future(s) with extra logic?

If yes, then you're SOL.  Read the long extensive docs.

If no, then you're probably just taking an existing future or stream, and adding some logic on top of it.
Great news!  You probably don't have to read the big long document.

First, make sure you can't just fulfill this use case with some method from FutureExt (e.g. `map` or `then`)
or StreamExt (make sure to read up on `unfold`).

Second, make sure you can't just fulfill this use case with some method from FutureExt or StreamExt.  I'm sorry,
I'm being patronizing, but that really does cover most cases.

Third, ok, you're determined to write your own impl, go grab the `pin-project-lite` crate.  You probably have
something like this...

```
pub struct MyStream<S: Stream> {
    stream: S,
    other_thing: SomeNotStreamStruct,
}
```

In your implementation of `poll_next` you probably want to call `poll_next` on `stream` (which means you'll
need to get it as `Pin<&mut S>` and you maybe want modify `other_thing` (which means you'll need to get it
as `&mut S`).  Guess what!  This is exactly what the `pin-project-lite` crate was designed for.  Just write...

```
#[pin_project]
pub struct FinallyStream<S: Stream, F: FnOnce()> {
    #[pin]
    stream: S,
    f: Option<F>,
}
```

For the fields that are inner stream / future things (e.g. `stream`) we mark it with `#[pin]`.  This means we
will get `Pin<&mut T>` and we can poll them.  For the things that aren't streams/futures we don't mark it with
`#[pin]`.  This means we will get `&mut T` and we can modify them.  All you need to do now is call
`let this = self.project()` in your `poll_next` function.

Let's look at a complete example.  I want to create a custom stream that calls some finalization method after
the stream has been exhuasted.  I use this in https://github.com/lancedb/lance to log query statistics after
a query has been polled to completion.  The complete implementation is here:

```
#[pin_project]
pub struct FinallyStream<S: Stream, F: FnOnce()> {
    #[pin]
    stream: S,
    f: Option<F>,
}

impl<S: Stream, F: FnOnce()> FinallyStream<S, F> {
    pub fn new(stream: S, f: F) -> Self {
        Self { stream, f: Some(f) }
    }
}

impl<S: Stream, F: FnOnce()> Stream for FinallyStream<S, F> {
    type Item = S::Item;

    fn poll_next(
        self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Option<Self::Item>> {
        let this = self.project();
        let res = this.stream.poll_next(cx);
        if matches!(res, std::task::Poll::Ready(None)) {
            // It's possible that None is polled multiple times, but we only call the function once
            if let Some(f) = this.f.take() {
                f();
            }
        }
        res
    }
}
```

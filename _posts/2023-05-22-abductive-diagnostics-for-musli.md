---
layout: post
title: Abductive diagnostics for Müsli
date: 2023-05-22 09:00 +0200
category: rust
---

Recently I released Müsli to the world. An [experimental binary serialization
framework]. And the response so far has been great. Clearly there's an interest
in improving the space.

Today I intend to dive into a specific topic I didn't manage to complete before
release. And it is a big one. **Error handling and diagnostics**.

[experimental binary serialization framework]: https://github.com/udoprog/musli

<!-- more -->

By the end of this post, I'll show you a design pattern which can give you rich
diagnostics like this:

```text
.field["hello"].vector[0]: should be smaller than 10, but was 42 (at bytes 34-36)
```

Which works for any Müsli format, has a small and easily optimized `Result::Err`
values, can be disabled with zero overhead, and provides full functionality in
`no_std` environments without an allocator.

If you find this topic interesting, [please join the discussion on Reddit](https://www.reddit.com/r/rust/comments/13okic7/abductive_diagnostics_for_m%C3%BCsli/).

Table of contents:

* [The problem](#the-problem)
* [The plan](#the-plan)
* [Abstractions that can melt away](#abstractions-that-can-melt-away)
* [Capturing errors](#capturing-errors)
* [I was promised no allocations](#i-was-promised-no-allocations)
* [Restoring what was lost](#restoring-what-was-lost)
* [Diagnostics](#diagnostics)
* [The cost of abstractions](#the-cost-of-abstractions)
* [Conclusion](#conclusion)

### The problem

Error handling seems fairly simple and given right? Rust has a `Result<T, E>`
which neatly implements the (unstable) [Try trait] so we can conveniently use
try expressions to propagate errors.

[Try trait]: https://doc.rust-lang.org/std/ops/trait.Try.html

```rust
#[derive(Decode)]
struct Object {
    #[musli(decode_with = decode_field)]
    field: u32,
}

fn decode_field<D>(decoder: D) -> Result<u32, D::Error>
where
    D: Decoder
{
    let value = decoder.decode_u32()?;

    if value >= 10 {
        return Err(D::Error::message(format_args!("should be smaller than 10, but was {value}"))));
    }

    Ok(value)
}
```

There are some of the *caveats* to this approach:

* The return type gets bigger, and is something that needs to be threaded by the
  Rust compiler all the way to the error handling facility.
* Where is the message stored? This implies that `D::Error` stores it somewhere
  ([sometimes cleverly]). If it's stored in a `Box<T>`, it needs to allocate. If
  it's stored inline in the type using [arrayvec] all return signatures now
  become larger by default.
* How do we know that the error was caused by `Object::field`? Should the error
  store the field name too? The name of the struct?
* How do we know what input caused the error? Do we also have to store a data
  location?

[sometimes cleverly]: https://docs.rs/anyhow/latest/anyhow/struct.Error.html
[arrayvec]: https://docs.rs/arrayvec/

These are all workable problems. In fact, [most serialization libraries]
enriches the diagnostics produced somehow so that it's more actionable.
Unfortunately I've found that you have to compromise between tight and complex
return values by putting everything in a box, or large error variants which have
a nasty habit of getting in the way of the compiler making things efficient.

Now say we want `no_std` without an allocator. Defining a rich error type is
starting to genuinely hurt - not to mention that it's pretty much unworkable due
to its size.

```rust
struct ErrorImpl {
    #[cfg(feature = "alloc")]
    message: String,
    #[cfg(not(feature = "alloc"))]
    message: ArrayString<64>,
    // The index where the error happened.
    index: usize,
}

#[cfg(feature = "alloc")]
use alloc::boxed::Box as MaybeBox;

#[cfg(not(feature = "alloc"))]
struct MaybeBox<T> {
    value: T,
}

struct Error {
    err: MaybeBox<ErrorImpl>,
}
```

So in `no_std` environments without `alloc` you tend to compromise. Let's look
at how a libraries provide diagnostics.

* [`serde_json`] tracks its own line and column using [by boxing the error].
* [`prost`] also provides diagnostics behind a box, [alongside the error
  description].
* [`postcard`] is a `no_std` binary serialization format for `serde`, and
  they've opted to [reduce actionable diagnostics] by only including error
  variants. This means no location diagnostics, but it also means smaller error
  types.

Well, I've had it with compromising. I want to try and fly a bit closer to the
sun and I'm bringing `no_std` with me.

[most serialization libraries]: https://docs.rs/serde_json/latest/src/serde_json/error.rs.html#175

[`serde_json`]: https://docs.rs/serde_json
[by boxing the error]: https://github.com/serde-rs/json/blob/7eeb169f9b51e2a30997d6c92aa3e170a2927b7f/src/error.rs#L175

[`prost`]: https://github.com/tokio-rs/prost
[alongside the error description]: https://github.com/tokio-rs/prost/blob/ad3a48c2f8a289fd023652a409578a753f867150/src/error.rs#L26

[`postcard`]: https://github.com/jamesmunns/postcard
[reduce actionable diagnostics]: https://docs.rs/postcard/latest/postcard/enum.Error.html

## The plan

In my [Müsli announcement] I tried to put emphasis on the experimental nature of
the framework. And the change I'm about to propose breaks the mold a little bit.
Maybe even a bit too much.

What I'll propose is a model for:

* Emitting dynamic errors and messages. Such as ones produces through
  `format_args!()`.
* Errors can track complex structural diagnostics. Such as which byte offset
  caused the error or even which field of a struct or index in an array caused
  it.
* All features are compile-time optional. Not by using features or `--cfg`
  flags, but through generics. In a single project you can freely mix and match
  which uses what. And you only pay for what you use.
* The pattern works in `no_std` environments without an allocator, such as micro
  controllers.

Sounds exciting right? To start things off I briefly want to touch on more
generally what it means to build abstractions in Rust that can melt away.

[Müsli announcement]: https://www.reddit.com/r/rust/comments/13ksqp2/comment/jkm038z/?utm_source=reddit&utm_medium=web2x&context=3

## Abstractions that can melt away

Let's imagine for a second we write a function like this to download a
collection of URLs:

```rust
fn download_urls<D, I>(downloader: D, urls: I) -> Result<Vec<Website>, Error>
where
    D: Downloader,
    I: IntoIterator<Item = Url>,
{
    let mut vec = Vec::new();

    for website in urls {
        vec.push(downloader.download(url)?);
    }

    Ok(vec)
}
```

Assuming that `urls` always contains some items, is there some way we can make
this function *conditionally* become a no-op depending on the implementation of
`D`? That is to say, Rust could correctly decide to remove most of the work.
Right now that doesn't seem to be the case - we are obligated to iterate over
the input and somehow produce one `Website` instance for each url
provided[^mock].

[^mock]: `Website` could support mocking. But maybe that's not viable either.

But with some minor changes we can build an *abstraction* that can be optimized
away, or in this case to the bare minimum of consuming the iterator[^effect]. We
make more elements of the function generic over the abstraction. Like this:

[^effect]: Since that is a legitimate effect we should preserve of calling the function.

```rust
trait Downloader {
    type Vec;

    fn new_vec() -> Self::Vec;

    fn download_into(url: Url, output: &mut Self::Vec) -> Result<(), Error>;
}

fn download_urls<D, I>(downloader: D, urls: I) -> Result<D::Vec, Error>
where
    D: Downloader,
    I: IntoIterator<Item = Url>,
{
    let mut vec = D::new_vec();

    for website in urls {
        downloader.download_into(url, &mut vec)?;
    }

    Ok(vec)
}
```

There's obviously more than one way to do this, but with this particular trait
we can easily build a `Downloader` which demonstrably does very little. It
doesn't even populate the vector. There *is* not even a vector.

```rust
struct DoNothing;

impl Downloader for DoNothing {
    type Vec = ();

    fn new_vec() {}

    fn download_into(url: Url, output: &mut ()) -> Result<(), Error> {
        Ok(())
    }
}
```

What I hope to demonstrate is that with a bit of inversion of control we can
make it easier for Rust to prove that nothing is going on and simply remove the
code. Here by moving implementation details into the `Downloader` trait.[^free]

[^free]: This isn't something that comes for free, we now have to live with the
    cognitive load of dealing with a weird interface. I think that most of the
    time it might not even be worth it.

## Capturing errors

Let's try and apply this idea to error handling by bringing back our friendly
`decode_field` function from above.

```rust
fn decode_field<D>(decoder: D) -> Result<u32, D::Error>
where
    D: Decoder
{
    let value = decoder.decode_u32()?;

    if value >= 10 {
        return Err(D::Error::message(format_args!("should be smaller than 10, but was {value}"))));
    }

    Ok(value)
}
```

So what if we want to collect the error message without allocating space for the
string directly in the error being returned?

To collect it we'd first need somewhere to put it, so let's introduce a trait
for it called `Context`.

```rust
trait Context {
    /// Report a message.
    fn message<T>(&mut self, message: T)
    where
        T: fmt::Display;
}
```

And as is appropriate, we now have to actually pass in an instance of a
`Context` that we can give our message to:

```rust
struct Error;

fn decode_field<C, D>(cx: &mut C, decoder: D) -> Result<u32, Error>
where
    C: Context,
    D: Decoder
{
    let value = decoder.decode_u32()?;

    if value >= 10 {
        cx.message(format_args!("should be smaller than 10, but was {value}"));
        return Err(Error);
    }

    Ok(value)
}
```

We're already starting to get somewhere interesting. The error variant is now a
zero sized type, which reduces the return size as much as possible. But what
exactly is a `Context`?

Since it's a trait the caller is responsible for providing an implementation. We
need it to somehow capture the reported message. One way to do it is to pack it
into an `Option<String>`.

```rust
#[derive(Default)]
struct Capture {
    message: Option<String>,
}

impl Context for Capture {
    fn message<T>(&mut self, message: T)
    where
        T: fmt::Display
    {
        self.message = Some(message.to_string());
    }
}
```

Converting a captured message back into the original error using this context
implementation is pretty straight forward:

```rust
let decoder = /* .. */;
let mut cx = Capture::default();

let Ok(value) = decode_field(&mut cx, decoder) else {
    return Err(D::Error::message(cx.message.unwrap()));
};
```

Do you see what's going on? All of our error handling and diagnostics -
regardless of what it is can be passed out through a pointer to the context.
This is why I call the pattern "abductive diagnostics", because the context
argument effectively abducts the error from the function.

But it's not for free. The cost we've imposed on our project is that the context
variable needs to be threaded through *every* fallible function which needs to
use it (something an [effect system] in Rust might someday remedy).

[effect system]: https://boats.gitlab.io/blog/post/the-problem-of-effects/

To improve on the cost / benefit of this pattern, let's add more information to
the context:

```rust
trait Context {
    /// indicate that n bytes of input has been processed.
    fn advance(&mut self, n: usize);
}
```

And with that extend our `Capture` to keep track of this:

```rust
#[derive(Default)]
struct Capture {
    message: Option<(usize, String)>,
    at: usize,
}

impl Context for Capture {
    fn advance(&mut self, n: usize) {
        self.at = self.at.wrapping_add(n);
    }

    fn message<T>(&mut self, message: T)
    where
        T: fmt::Display
    {
        self.message = Some((self.at, message.to_string()));
    }
}
```

Now we can associate byte indexes to diagnostics. We're really starting to build
out our capabilities![^advance] The neat part here is that we added some really
powerful capabilities to our system, while keeping the returned error a zero
sized type.

[^advance]: We do need to make sure to call `advance` where appropriate, like
    for [every procedure which reads input].

[every procedure which reads input]: https://github.com/udoprog/musli/blob/80fd962eb4a762642ca7a3c70af1aa7652519693/crates/musli-common/src/reader.rs#L193

Next let's see how this pattern can help us to capture errors *without*
allocating on the heap.

## I was promised no allocations

Right now there's clearly a `String` there, which uses allocations! It's even
[in the `alloc` crate].

This is true, but the neat part about having the context abduct our errors is
that we gain the necessary control to store them wherever we want.

So let's build a context which stores errors on the stack instead using
[`arrayvec`].[^fuzzy]

[in the `alloc` crate]: https://doc.rust-lang.org/alloc/string/struct.String.html
[`arrayvec`]: https://docs.rs/arrayvec

[^fuzzy]: The exact behavior of writing to an `ArrayString` at capacity is a bit
    fuzzy, but at the very least anything that is outside of its capacity will
    be drop. [The exact boundary of which could be improved].

[The exact boundary of which could be improved]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=7b95a0a92730ae701269e71115c43aab

```rust
use std::fmt::Write;

use arrayvec::ArrayString;

#[derive(Default)]
struct Capture<const N: usize> {
    message: ArrayString<N>,
    message_at: usize,
    at: usize,
}

impl<const N: usize> Context for Capture<const N: usize> {
    fn advance(&mut self, n: usize) {
        self.at = self.at.wrapping_add(n);
    }

    fn message<T>(&mut self, message: T)
    where
        T: fmt::Display
    {
        use std::fmt::Write;
        self.message_at = self.at;
        self.message.clear();
        let _ = write!(&mut self.message, "{}", message);
    }
}
```

We can imagine all kinds of ways for storing errors. Müsli comes out of the box
with two that uses different strategies:

* One which allocates, in [`AllocContext`].
* One which stores errors and diagnostics on the stack, in [`NoStdContext`].

But if you intend to integrate it into a strange environment you would very much
be encouraged to implement your own `Context`.

[`AllocContext`]: https://github.com/udoprog/musli/blob/be09b1fedef1a4040a1b8028b3eaa714482dc48b/crates/musli-common/src/context/alloc_context.rs#L34
[`NoStdContext`]: https://github.com/udoprog/musli/blob/be09b1fedef1a4040a1b8028b3eaa714482dc48b/crates/musli-common/src/context/no_std_context.rs#L30

## Restoring what was lost

If we pay attention to the method we refactored above, we might note that while
we gained the ability to *abduct errors* through the context, we lost two things
as well.

```rust
struct Error;

fn decode_field<C, D>(cx: &mut C, decoder: D) -> Result<u32, Error>
where
    C: Context,
    D: Decoder
{
    let value = decoder.decode_u32()?;

    if value >= 10 {
        return Err(Error);
    }

    Ok(value)
}
```

Do you see it?

* Regular errors which contains their own diagnostics can no longer be returned
  if we wanted to; and
* The method *doesn't guarantee* that an error has been reported to the context
  **oops**.

The latter is no good. When using regular error types we're forced to *somehow*
produce an `Error` through some kind of constructor. Here we can just return the
marker type and in the worst case forget to provide a message.

So let's address this by modifying `Context` further:

```rust
trait Context {
    /// The error type produced by the context.
    type Error;

    /// Add ways to construct a `Self::Error`.
    fn message<T>(&mut self, message: T) -> Self::Error
    where
        T: fmt::Display;
}
```

And the corresponding changes to `decode_field` looks like this:

```rust
fn decode_field<C, D>(cx: &mut C, decoder: D) -> Result<u32, C::Error>
where
    C: Context,
    D: Decoder
{
    let value = decoder.decode_u32()?;

    if value >= 10 {
        return Err(cx.message(format_args!("should be smaller than 10, but was {value}")));
    }

    Ok(value)
}
```

Now the only way we can return from `decode_field` is by either producing
`Ok(u32)`, or `Err(C::Error)`. And the `Context` is the only one which can
produce `C::Error`'s, so we don't accidentally return a blank error marker
without providing diagnostics.

In addition, do you remember that the `Decoder` also produces an error? The call
to `decode_u32` doesn't actually compile. We have to handle that somehow as
well. To do this, we extend our context further:

```rust
type Context {
    type Input;
    type Error;

    /// Add the ability to report an error that can be converted to `Input`.
    fn report<T>(&mut self, input: T) -> Self::Error
    where
        Self::Input: From<T>;
}
```

We can now specify the `Input` type as the deserializer error that the context
can abduct:

```rust
fn decode_field<C, D>(cx: &mut C, decoder: D) -> Result<u32, C::Error>
where
    C: Context<Input = D::Error>,
    D: Decoder
{
    let value = decoder.decode_u32().map_err(|err| cx.report(err))?;

    if value >= 10 {
        return Err(cx.message(format_args!("should be smaller than 10, but was {value}")));
    }

    Ok(value)
}
```

Of course in the real implementation we just pass along the `cx` variable to
`decode_u32`. But this showcases how the pattern can be gradually introduced
into existing code which was of great help during refactoring.

What exactly the `report` implementation looks like I leave as an exercise to
the reader, but with these additions there are now two more interesting contexts
that Müsli provides:

* [`Same`] which produces the same error (`Context::Error`) that it consumes
  (`Context::Input`) providing full backwards compatibility.
* [`Ignore`] which simply records that an error has happened, but returns a zero
  sized marker type like before.

[`Same`]: https://github.com/udoprog/musli/blob/80fd962/crates/musli-common/src/context.rs#L31
[`Ignore`]: https://github.com/udoprog/musli/blob/80fd962/crates/musli-common/src/context.rs#L75

```rust
// Behaves just like the original version without `Context` support.
let mut cx = Same::default();
let value = decode_field(&mut cx, decoder)?;
```

```rust
// The error returned is a ZST.
let mut cx = Ignore::default();
let Ok(value) = decode_field(&mut cx, decoder) else {
    // deal with the fact that something went wrong.
};
```

## Diagnostics

Error messages by themselves are cool and all, but what we really want is more
*diagnostics*. While it can be useful to know that something `should be smaller
than 10, but was 42` this only helps us in the simplest cases to troubleshoot
issues. In most cases we don't just need to know *what* went wrong, but *where*
it went wrong.

To this end we can add more hooks to `Context`. And this is where it starts
getting really interesting. In Müsli I've opted to add support for keeping track
of the byte index and fully tracing the type hierarchy being decoded.

```rust
trait Context {
    fn advance(&mut self, n: usize);

    fn enter_struct(&mut self, name: &'static str);
    fn leave_struct(&mut self);

    fn enter_enum(&mut self, name: &'static str);
    fn leave_enum(&mut self);

    fn enter_named_field<T>(&mut self, name: &'static str, tag: T)
    where
        T: fmt::Display;
    fn enter_unnamed_field<T>(&mut self, index: u32, tag: T)
    where
        T: fmt::Display;
    fn leave_field(&mut self);

    fn enter_variant<T>(&mut self, name: &'static str, tag: T)
    where
        T: fmt::Display;
    fn leave_variant(&mut self);

    fn enter_map_key<T>(&mut self, field: T)
    where
        T: fmt::Display;
    fn leave_map_key(&mut self);

    fn enter_sequence_index(&mut self, index: usize);
    fn leave_sequence_index(&mut self);
}
```

So using one of the contexts (like `AllocContext`) provided above, we can
get error messages like this:

```text
.field["hello"].vector[0]: should be smaller than 10, but was 42 (at byte 36)
```

Or even this, which includes the names of the Rust types which were involved:

```text
Struct { .field["hello"] = Enum::Variant2 { .vector[0] } }: should be smaller than 10, but was 42 (at byte 36)
```

Or nothing at all! If you don't use it, you don't pay for it:

```text
decoding / encoding failed
```

Conveniently our [`Encode` and `Decode` derives] fills out all these context
calls for you. So tracing is something you get for free[^details].

[^details]: There are more interesting details [in the full implementation of `Context`].

[in the full implementation of `Context`]: https://github.com/udoprog/musli/blob/main/crates/musli/src/context.rs
[`Encode` and `Decode` derives]: https://docs.rs/musli/latest/musli/derives/

## The cost of abstractions

I'm keenly aware that threading the context variable through the entire
framework can be a bit messy. It took me almost two days to refactor all the
code in Müsli. In the long run the ergonomics of it might simply pan out to not
be worth it, or we'll try to change something to make it easier. For now I don't
know what.

Luckily most users will not interact with the plumbing of the framework. They
should primarily focus on using the high level [`derives`] available. A smaller
number of users will end up writing hooks and that can be harder because there's
a lot of things going on:

[`derives`]: https://docs.rs/musli/latest/musli/derives/index.html

```rust
use musli::{Context, Mode, Encoder, Decoder};
    
struct MyType {
    /* .. */
}

fn encode<'buf, M, E, C>(my_type: &MyType, cx: &mut C, encoder: E) -> Result<E::Ok, C::Error>
where
    M: Mode,
    C: Context<'buf, Input = E::Error>,
    E: Encoder,
{
    todo!()
}

fn decode<'de, 'buf, M, C, D>(cx: &mut C, decoder: D) -> Result<MyType, C::Error>
where
    M: Mode,
    C: Context<'buf, Input = D::Error>,
    D: Decoder<'de>,
{
    todo!()
}
```

Two lifetimes, and three generics by default, and a strange `Input` parameter
for `Context`! Did I mention that Müsli is still an experimental project?

Still, I'm hoping to discover a better way of doing this from an ergonomics
perspective. If you have suggestions, I'd love to hear them!

## Conclusion

Thank you for reading!

I can confidently say that with this, Müsli has the ability to produce state of
the art diagnostics with almost no effort from the user. I think most people
who've worked with serialization knows that without good diagnostics it can be
very hard to figure out what's going on.

I will round off by displaying this error I'm currently plagued by, which is
caused by a binary format using `serde`. It's actually the reason why I started
to pursue this topic:

```text
Error: ../../changes.gz

Caused by:
    invalid value: integer `335544320`, expected variant index 0 <= i < 11
```

I don't know which field, in which struct, being fed what data, caused that
error to happen. Needless to say I'll be switching that project to use Müsli.

The above certainly can be improved, and we can even add tracing to `serde`. But
not without compromises. And I want everything.

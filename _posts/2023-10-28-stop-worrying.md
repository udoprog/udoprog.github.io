---
layout: post
title: How I learned to stop worrying and love byte ordering
date: 2023-10-28 06:00 +0200
category: rust
---

In the [last post] I announced [`musli-zerocopy`]. In this one I'll discuss one
of the concerns you'll need to mitigate to build and distribute archives:
endianness.

[last post]: {% post_url 2023-10-19-musli-zerocopy %}

<!-- more -->

There's broadly two distinct ways that we organize numerical types in memory. We
either store the least significant byte first in memory, known as "little
endian", or we store them last calling this "big endian". This is known as the
[*endianness*] or *byte order* of a value in memory.

[*endianness*]: https://en.wikipedia.org/wiki/Endianness

```
0x42 0x00 0x00 0x00 // 0x42u32 little endian
0x00 0x00 0x00 0x42 // 0x42u32 big endian
```

Your system has a native byte order. This is the way it expects data to be
organized in memory, so when you're accessing a value like:

```rust
let value = 42u32;
dbg!(value);
```

The value `42` above is stored on the stack using one of the above layouts. Most
likely little endian.

<br>

The first version of [`musli-zerocopy`] ignores byte orders. You'd be using the
byte order that is native to the platform you're on. This is not much of a
problem for simple structures, since primitive fields in simple structs like
`u32` natively can represent data in any byte order. Rust's `std` library comes
with primitives for [handling conversion], which in byte-order parlance is known
as "swapping".

```rust
#[repr(C)]
struct Archive {
    field: u32,
    field2: u64,
}

let archive = Archive {
    field: 42u32.to_be(),
    field2: 42u64.to_le(),
}
```

> A struct living in memory which quite happily uses a mixed byte order across
> fields.

Now if you don't know the byte order of the fields in the archive. Reading it
would be surprising:

```rust
dbg!(archive.field); // 704643072
dbg!(archive.field2); // 42
```

We get this value (`0x2a000000`, since `42` is `0x2a`) because of the mismatch
between the native byte order, and the byte order of the the field. To read it
correctly, we need to swap the bytes back. All of these achieve that result on a
little-endian system:

```rust
dbg!(u32::from_be(archive.field)); // 42
dbg!(archive.field.swap_bytes()); // 42
dbg!(archive.field.to_be()); // 42
```

> It's worth pondering why `u32::to_be` works just as well. There's no
> *inherent* byte order in the value being swapped. All a method like `to_be`
> does is convert to a no-op in case the native byte order is already
> big-endian. Otherwise it performs the same swapping operation.

[`musli-zerocopy`]: https://crates.io/crates/musli-zerocopy
[handling conversion]: https://doc.rust-lang.org/std/primitive.u32.html#method.from_be

I hope this makes it apparent why a byte-order *naive* application can't process
archives correctly unless the byte order used in the archive matches the native
byte order. In order to interpret the values correctly we somehow need extra
information and careful processing.

<br>

[The dictionary] I built this library for demands that archives are distributed.
So one way or it has to be portable with byte ordering in mind. In this post
I'll detail high level strategies to deal with byte ordering, and introduce the
tools I've added to `musli-zerocopy` to make it - if not easy - manageable to
work with.

[The dictionary]: https://github.com/udoprog/jpv

Overview:
* [**Why even bother?**](#why-even-bother)
* [**Option #1: Don't bother**](#option-1-dont-bother)
* [**Option #2: Care a lot**](#option-2-care-a-lot)
* [**Option #3: Why not both?**](#option-3-why-not-both)
* [**Option #4: Squint hard enough and everything looks like an optimization**](#option-4-squint-hard-enough-and-everything-looks-like-an-optimization)
* [**Detour: A missed enum optimization**](#detour-a-missed-enum-optimization)

<br>

## Why even bother?

Let me make a prediction: Your system uses little endian. This is a good
prediction, because *little endian* has essentially won. One reason it's a great
choice is because converting between wide to narrow types is easy because their
arrangement overlaps. To go from a `u32` to a `u16` you simply just read the
first two bytes of the `u32` easy peasy.

```
0x42 0x00 0x00 0x00 // 0x42u32
0x42 0x00 ---- ---- // 0x42u16
```

None of the platforms I intend to distribute my dictionary too are big-endian.
Yet it's still important to make sure that it works:

* You don't want to end up relying on abstractions or designs which end up being
  a dead end or at worst cause bit rot down the line. *Just in case*.
* Some formats impose arbitrary ordering demands. I'd like to be able to support
  these.
* Big-endian [for *reasons*] ended up being the de-facto byte order used in
  network protocols.

[for *reasons*]: https://stackoverflow.com/questions/13514614/why-is-network-byte-order-defined-to-be-big-endian

But why is it good for values to be *natively* byte ordered?

Rust can generate significantly improved code when things align well with the
current platform. The following is a difference between two functions - one
which loads an `[u32; 8]` array using `u32`'s in its native byte order
(effectively copying it), and the other converted using a non-native byte order:

```rust
use musli_zerocopy::endian;

pub fn convert_le(values: [u32; 8]) -> [u32; 8] {
    endian::from_le(values)
}

pub fn convert_be(values: [u32; 8]) -> [u32; 8] {
    endian::from_be(values)
}
```

The two functions result in the following difference in assembly:

```diff
  mov     rax, rdi
- movups  xmm0, xmmword, ptr, [rsi]
- movups  xmm1, xmmword, ptr, [rsi, +, 16]
- movups  xmmword, ptr, [rdi, +, 16], xmm1
- movups  xmmword, ptr, [rdi], xmm0
+ movdqu  xmm0, xmmword, ptr, [rsi]
+ movdqu  xmm1, xmmword, ptr, [rsi, +, 16]
+ pxor    xmm2, xmm2
+ movdqa  xmm3, xmm0
+ punpckhbw xmm3, xmm2
+ pshuflw xmm3, xmm3, 27
+ pshufhw xmm3, xmm3, 27
+ punpcklbw xmm0, xmm2
+ pshuflw xmm0, xmm0, 27
+ pshufhw xmm0, xmm0, 27
+ packuswb xmm0, xmm3
+ movdqa  xmm3, xmm1
+ punpckhbw xmm3, xmm2
+ pshuflw xmm3, xmm3, 27
+ pshufhw xmm3, xmm3, 27
+ punpcklbw xmm1, xmm2
+ pshuflw xmm1, xmm1, 27
+ pshufhw xmm1, xmm1, 27
+ packuswb xmm1, xmm3
+ movdqu  xmmword, ptr, [rdi, +, 16], xmm1
+ movdqu  xmmword, ptr, [rdi], xmm0
  ret
```

Despite some neat vectorization, there's still a significant difference in code
size and execution complexity. Work is work, and work that can be avoided means
faster applications that use less power.

So how do we accomplish this? In the next four sections I'll cover some of the
available options.

<br>

## Option #1: Don't bother

The documentation in [`musli-zerocopy` (Reading data)] mentions that there are some pieces of
data which are needed to transmit an archive from one system to another, we call
it its DNA (borrowed from [Blender]), these are:

[`musli-zerocopy` (Reading data)]: https://docs.rs/musli-zerocopy/latest/musli_zerocopy/#reading-data

* The alignment of the buffer.
* The byte order of the system which produced the buffer.
* If pointer-sized types are used, the size of a pointer.
* The structure of the type being read.

[Blender]: https://wiki.blender.jp/Dev:Source/Architecture/File_Format#File-Header

One option is to include a portable header which indicates its format and if
there is a mismatch we refuse to read the archive and [`bail!`] with an error.
We want to avoid the possibility of the application "accidentally" working since
this might cause unexpected behaviors. Unexpected behaviors at best causes an
error later down the line, at worst a logic bug leading to a security issue.

> If *security* is an application concern, additional safeguards should be put
> into place, such as integrity checking. But this is not unique to zero-copy
> serialization.

[`bail!`]: https://docs.rs/anyhow/latest/anyhow/macro.bail.html

```rust
use anyhow::bail;

use musli_zerocopy::ZeroCopy;
use musli_zerocopy::endian;

/// A marker type for the archive indicating its endianness.
///
/// Since this is a single byte, it is already portable.
#[derive(ZeroCopy)]
#[repr(u8)]
enum Endianness {
    Big,
    Little,
}

/// The header of the archive.
#[derive(ZeroCopy)]
#[repr(C)]
struct Header {
    endianness: Endianness,
    pointer_size: u32,
    alignment: u32,
}

let buf: &Buf = /* .. */;

let header = buf.load_at::<Header>(0)?;

if header.endianness != endian::pick!(Endianness::Big, Endianness::Little) {
    bail!("Archive has the wrong endianness");
}
```

The [`pick!` macro] is a helper provided by `musli-zerocopy`. It will evaluate
to the first argument on big endian systems, and the second on little endian
systems. We'll make more use of this below.

[`pick!` macro]: https://docs.rs/musli-zerocopy/latest/musli_zerocopy/endian/macro.pick.html

This is a nice alternative in case your application can download a platform
suitable archive directly. Its DNA can then be represented as a target string,
like `dictionary-jp-64-be.bin`.

<br>

## Option #2: Care a lot

This option is a bit more intrusive, but allows for an archive to be used even
if there is a DNA mismatch. We can interact with the archive, but systems with a
mismatching byte order will simply have to take the performance hit.

In `musli-zerocopy` this is done by explicitly setting endianness and using
wrapping primitives like `Endian<T, E>`:

```rust
use musli_zerocopy::{Ref, Endian};
use musli_zerocopy::endian::Little;

#[derive(ZeroCopy)]
#[repr(C)]
struct Data
where
    E: ByteOrder,
{
    name: Ref<str, Little>,
    age: Endian<u32, Little>,
}
```

Reading the field can only be done through accessors such as `Endian::to_ne`.
Calling the method will perform the conversion in-place.

```rust
use musli_zerocopy::Buf;

let buf = &Buf = /* .. */;

let data: &Data = buf.load_at(0)?;

assert_eq!(data.age.to_ne(), 35);
```

It isn't strictly necessary to make use of endian-aware primitives in
`musli-zerocopy`, since most numerical types can inhabit all reflexive bit
patterns. It could simply be declared like this and force the bit pattern using
one of the provided conversion methods such as `endian::from_le`.

```rust
use musli_zerocopy::{Buf, Ref, Endian};
use musli_zerocopy::endian;

#[derive(Clone, Copy, ZeroCopy)]
#[repr(C)]
struct Data
where
    E: ByteOrder,
{
    name: Ref<str>,
    age: u32,
}

let buf = &Buf = /* .. */;

let data: &Data = buf.load_at(0)?;

assert_eq!(endian::from_le(data.age), 35);
```

<br>

## Option #3: Why not both?

Maybe the most interesting alternative of them all. Store the archive in
multiple orderings and read the one which matches your platform.

🤯

In musli we can do this by storing two references in the header, and make use of
the `pick!` macro to decide which one to use. Due to the default value of
`Data<E = Native>` the correct byte order is used intrinsically after it's been
identified since `Native` is simply an alias to the system ordering.

```rust
use musli_zerocopy::{ByteOrder, Endian, Ref, ZeroCopy};
use musli_zerocopy::endian::{Big, Little, Native};

#[derive(ZeroCopy)]
#[repr(C)]
struct Header {
    big: Ref<Data<Big>, Big>,
    little: Ref<Data<Little>, Little>,
}

#[derive(ZeroCopy)]
#[repr(C)]
struct Data<E = Native>
where
    E: ByteOrder,
{
    name: Ref<str, E>,
    age: Endian<u32, E>,
}

let buf = &Buf = /* .. */;

let data: &Header = buf.load_at(0)?;
let data: Ref<Data> = endian::pick!(header.big, header.little);

println!("Name: {}", buf.load(data.name)?);
println!("Age: {}", *data.age);
```

On the up, we get to interact with natively ordered data. On the down, we have
to store two versions of every byte-order sensitive data structure in the
archive. I took great care to ensure that this can be done generically in Müsli
so you at least only need to write the code which populates the archive once.

```rust
use musli_zerocopy::{ByteOrder, Endian, OwnedBuf};

fn build_archive<E: ByteOrder>(buf: &mut OwnedBuf) -> Data<E> {
    let name = buf.store_unsized("John Doe");

    Data {
        name,
        age: Endian::new(35),
    }
}

let big = build_archive(&mut buf);
let little = build_archive(&mut buf);

let header = Header {
    big,
    little,
};
```

The keen eyed might note that this is a very typical time-memory trade off. We
provide the perfect representation for every platform at the cost of increasing
the space used.

In theory we could provide one per pointer-size as well, but the number of
permutations start to become ... unwieldy.

> This is a similar approach to [multi-architecture
> binaries](https://en.wikipedia.org/wiki/Fat_binary) which are supported on
> Apple.

<br>

## Option #4: Squint hard enough and everything looks like an optimization

This strategy involves converting a stable well-defined archive into an
optimized zero-copy archive that can be loaded on subsequent invocations of your
applications. It does mean "taking the hit" if the local archive needs to be
constructed, but once it has been you've got the best of all worlds.

This is how we'll go about it:

* Use one of [Müsli's portable formats] to store the original archive.
* First time the archive is opened, convert it into a zero-copy archive with a
  native optimized structure and store a local copy.
* Load the zero-copy archive.

This is the final use-case [I intend to improve support for in
`musli-zerocopy`]. But even without explicit support it is rather straight
forward to implement.

[Müsli's portable formats]: https://docs.rs/musli/latest/musli/#formats
[I intend to improve support for in `musli-zerocopy`]: https://github.com/udoprog/musli/issues/49

```rust
use anyhow::Context;

use std::fs;
use std::path::Path;

use musli::Decode;
use musli_zerocopy::{Ref, Buf, ZeroCopy};
use musli_zerocopy::swiss;

#[derive(Decode)]
struct Archive {
    map: HashMap<String, u32>,
}

#[derive(ZeroCopy)]
#[repr(C)]
struct ArchiveNative {
    map: swiss::MapRef<Ref<str>, u32>,
}

let archive = Path::new("archive.bin");
let archive_native = Path::new("archive-native.bin");

if !archive_native.is_file() || is_outdated(archive, archive_native) {
    tracing::trace!("Missing or outdated {}, building {}", archive.display(), archive_native.display());

    let archive = fs::read(archive).context("reading archive")?;
    let archive: Archive = musli_storage::from_slice(&archive).context("decoding archive")?;
    write_native_archive(&archive, archive_native).context("building native archive")?;
}

let buf: &Buf = Buf::new(mmap_archive(archive_native));
```

One might be tempted to call this the best of all worlds, as long as you're OK
with spending the disk space for it.

This can even be combined with [Option #2](#option-2-care-a-lot), but we only
bother converting and re-saving the archive if it is the wrong byte order.

<br>

## Detour: A missed enum optimization

The goal with these APIs are to ensure that any endian-aware operations impose
zero cost if the demanded byte order matches the current one. We have all [the
necessary tools] to muck about with the bytes of a type. Any native-to-native
swapping be they generic or not should result in a simple move, like this:

[the necessary tools]: https://docs.rs/musli-zerocopy/latest/musli_zerocopy/endian/index.html

```rust
use musli_zerocopy::{endian, ZeroCopy};

#[derive(ZeroCopy)]
#[repr(C)]
pub struct Struct {
    bits32: u32,
    bits64: u64,
    bits128: u128,
    array: [u8; 16],
}

#[inline(never)]
pub fn ensure_struct_le(st: Struct) -> Struct {
    st.swap_bytes::<endian::Little>()
}

#[inline(never)]
pub fn ensure_struct_be(st: Struct) -> Struct {
    st.swap_bytes::<endian::Big>()
}
```

The generated assembly for each function follows:

```nasm
ensure_struct_le:
 mov     rax, rdi
 movups  xmm0, xmmword, ptr, [rsi]
 movups  xmm1, xmmword, ptr, [rsi, +, 16]
 movups  xmm2, xmmword, ptr, [rsi, +, 32]
 movups  xmmword, ptr, [rdi, +, 32], xmm2
 movups  xmmword, ptr, [rdi, +, 16], xmm1
 movups  xmmword, ptr, [rdi], xmm0
 ret
```

> In little-endian swapping a struct amounts to copying it. This is in fact the
> same code that we'd generate if we just passed the struct through.

```nasm
ensure_struct_be:
 mov     rax, rdi
 mov     ecx, dword, ptr, [rsi]
 mov     rdx, qword, ptr, [rsi, +, 8]
 mov     rdi, qword, ptr, [rsi, +, 24]
 mov     r8, qword, ptr, [rsi, +, 16]
 bswap   r8
 bswap   rdi
 bswap   rdx
 movups  xmm0, xmmword, ptr, [rsi, +, 32]
 bswap   ecx
 mov     dword, ptr, [rax], ecx
 mov     qword, ptr, [rax, +, 8], rdx
 mov     qword, ptr, [rax, +, 16], rdi
 mov     qword, ptr, [rax, +, 24], r8
 movups  xmmword, ptr, [rax, +, 32], xmm0
 ret
```

> On a big-endian systems we have to do the work of swapping each field while
> copying it.

<br>

Overall this process has proven to be quite smooth. With the exception for one
instance. Consider the following function:

```rust
pub enum Enum {
    Empty,
    One(u32),
    Two(u32, u64),
}

pub fn move_enum(en: Enum) -> Enum {
    match en {
        Enum::Empty => Enum::Empty,
        Enum::One(first) => Enum::One(first),
        Enum::Two(first, second) => Enum::Two(first, second),
    }
}
```

This should be possible to translate to a simple move, where the bits of the
enum are copied. Essentially what we expect to see is this:

```nasm
move_enum:
 mov     rax, rdi
 movups  xmm0, xmmword, ptr, [rsi]
 movups  xmmword, ptr, [rdi], xmm0
 ret
```

Rather worryingly what we get instead is this:

```nasm
move_enum:
 mov     rax, rdi
 mov     ecx, dword, ptr, [rsi]
 test    rcx, rcx
 je      .LBB0_1
 cmp     ecx, 1
 jne     .LBB0_4
 mov     ecx, dword, ptr, [rsi, +, 4]
 mov     dword, ptr, [rax, +, 4], ecx
 mov     ecx, 1
 mov     dword, ptr, [rax], ecx
 ret
.LBB0_1:
 xor     ecx, ecx
 mov     dword, ptr, [rax], ecx
 ret
.LBB0_4:
 mov     ecx, dword, ptr, [rsi, +, 4]
 mov     rdx, qword, ptr, [rsi, +, 8]
 mov     dword, ptr, [rax, +, 4], ecx
 mov     qword, ptr, [rax, +, 8], rdx
 mov     ecx, 2
 mov     dword, ptr, [rax], ecx
 ret
```

> [Try it for yourself on godbolt](https://godbolt.org/z/Pno56Gsnr).

Rust decides to check the discriminant and generate a branch for each *unique*
variant. It then proceeds to carefully just copy the data for that variant.

This is a unfortunate for us since we use this pattern when byte swapping enums.
If we did nothing, it would mean that native-to-native swapping wouldn't be a
no-op:

```rust
use musli_zerocopy::{ByteOrder, ZeroCopy};

pub enum Enum {
    Empty,
    One(u32),
    Two(u32, u64),
}

impl ZeroCopy for Enum {
    fn swap_bytes<E: ByteOrder>(en: Enum) -> Enum {
        match en {
            Enum::Empty => Enum::Empty,
            Enum::One(first) => Enum::One(first.swap_bytes::<E>()),
            Enum::Two(first, second) => Enum::Two(first.swap_bytes::<E>(), second.swap_bytes::<E>()),
        }
    }
}
```

Why this happens is a rather [unfortunate case of a missed optimization]. It
never got merged due to fears of a potential performance regression and wasn't
picked up. Hopefully these types of MIR-based optimizations can be revived
again, because without them we still might worry about reducing "LLVM bloat" to
improve compile times over writing idiomatic Rust.

[unfortunate case of a missed optimization]: https://github.com/rust-lang/rust/pull/69756

```rust
// Avoid `Option::map` because it bloats LLVM IR.
match self.remove_entry(k) {
    Some((_, v)) => Some(v),
    None => None,
}
```

> Example form [`rust-lang/hashbrown`](https://github.com/rust-lang/hashbrown/blob/778e235eccc233f2d8b906ae8e4c3d717c44fcce/src/map.rs#L1312).

Once the issue was identified the fix was simple enough. [I've introduced a
method] on `ByteOrder` which optionally maps a value in case the byte order
differs from `Native` ensuring that native-to-native swapping doesn't do the
extra work dealing with each unique variant individually.

[I've introduced a method]: https://github.com/udoprog/musli/pull/48

<br>

## Conclusion

Congratulations on reading this far. I hope you've enjoying my explorations into
the zero-copy space.

If there's anything you want to ask or comment on, feel free to post below or
[on Reddit]. Thank you!

[on Reddit]: https://www.reddit.com/r/rust/comments/17i6txx/how_i_learned_to_stop_worrying_and_love_byte/

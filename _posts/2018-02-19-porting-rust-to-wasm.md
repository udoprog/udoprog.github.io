---
layout: post
title: "Porting Rust to WebAssembly"
description: ""
category: rust
---

I recently spent some effort trying to make [`reproto`] run in a browser.
Here I want to outline the problems I encountered and how I worked around them.
I will also provide a number of suggestions for how things might be improved for future porters.

[`reproto`]: https://github.com/reproto

<!-- more -->

A big chunk of the `wasm` story on Rust currently relies on [`stdweb`].

Needless to say, this project is _incredible_.
`stdweb` makes it smooth to build Rust applications that integrates with a browser.

There's a ton of things that can be said about that project, but first I want to focus on the porting efforts of [`reproto`].

[`stdweb`]: https://github.com/koute/stdweb
[`reproto`]: https://github.com/reproto

## Getting up to speed and shaking the tree

The best way to port a library is to compile it.

For Rust it's as simple as installing [`cargo-web`] and setting up a project with all your dependencies.

All it really needs is a [`Cargo.toml`] declaring your dependencies, and a [`src/main.rs`] with `extern`
declarations for everything you want compiled.

For `reproto`, that process looks like this:

```
# Cargo.toml

[package]
# skipped

[dependencies]
reproto-core = {path = "../lib/core", version = "0.3"}
reproto-compile = {path = "../lib/compile", version = "0.3"}
reproto-derive = {path = "../lib/derive", version = "0.3"}
reproto-manifest = {path = "../lib/manifest", version = "0.3"}
reproto-parser = {path = "../lib/parser", version = "0.3"}
reproto-backend-java = {path = "../lib/backend-java", version = "0.3"}
reproto-backend-js = {path = "../lib/backend-js", version = "0.3"}
reproto-backend-json = {path = "../lib/backend-json", version = "0.3"}
reproto-backend-python = {path = "../lib/backend-python", version = "0.3"}
reproto-backend-rust = {path = "../lib/backend-rust", version = "0.3"}
reproto-backend-reproto = {path = "../lib/backend-reproto", version = "0.3"}

stdweb = "0.3"
serde = "1"
serde_json = "1"
serde_derive = "1"
```

```rust
//! src/main.rs

extern crate serde;
#[macro_use]
extern crate serde_derive;
extern crate serde_json;
#[macro_use]
extern crate stdweb;

extern crate reproto_backend_java as java;
extern crate reproto_backend_js as js;
extern crate reproto_backend_json as json;
extern crate reproto_backend_python as python;
extern crate reproto_backend_reproto as reproto;
extern crate reproto_backend_rust as rust;
extern crate reproto_compile as compile;
extern crate reproto_core as core;
extern crate reproto_derive as derive;
extern crate reproto_manifest as manifest;
extern crate reproto_parser as parser;

fn main() {
    stdweb::initialize();
}
```

Finally we want to add a `Web.toml`, which will allow us to specify the default target so we won't
have to type it out all the time:

```toml
# Web.toml

default-target = "wasm32-unknown-unknown"
```

Now the project should build by running `cargo web build`.

For tracing down where dependencies come from, I relied heavily on [`cargo-tree`].

[`cargo-tree`]: https://github.com/sfackler/cargo-tree

When you do encounter a problem `cargo tree` can quickly determine how a given package was pulled
in:

```
cargo tree
[dependencies]
├── reproto-backend-java v0.3.13 (file://<home>/repo/reproto/lib/backend-java)
│   [dependencies]
│   ├── genco v0.2.6 (*)
│   ├── log v0.3.9
│   │   [dependencies]
│   │   └── log v0.4.1
│   │       [dependencies]
│   │       └── cfg-if v0.1.2
... snip
```

[`cargo-web`]: https://github.com/koute/cargo-web
[`Cargo.toml`]: https://github.com/reproto/reproto/blob/0356c90c60/wasm/Cargo.toml
[`src/main.rs`]: https://github.com/reproto/reproto/blob/0356c90c60/wasm/src/main.rs

## Big numbers

My project is structured into many different modules, each loosely responsible for one aspect of
the solution.

The first module where I encountered problems was [`core`].

[`core`]: https://github.com/reproto/reproto/blob/0356c90c60/lib/core/

The [`num`] crate _by default_ pulls in [`rustc-serialize`], which fails like this:

[`num`]: https://crates.io/crates/num
[`rustc-serialize`]: https://crates.io/crates/rustc-serialize

```
     |
853  |     fn encode<S: Encoder>(&self, s: &mut S) -> Result<(), S::Error>;
     |     ---------------------------------------------------------------- `encode` from trait
...
1358 | impl Encodable for path::Path {
     | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing `encode` in implementation

error[E0046]: not all trait items implemented, missing: `decode`
    --> <registry>/src/github.com-1ecc6299db9ec823/rustc-serialize-0.3.24/src/serialize.rs:1382:1
     |
904  |     fn decode<D: Decoder>(d: &mut D) -> Result<Self, D::Error>;
     |     ----------------------------------------------------------- `decode` from trait
...
1382 | impl Decodable for path::PathBuf {
     | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing `decode` in implementation
```

There appears to be a trait implementation missing.

Opening up the specified line (`1382`) reveals that rust-serialize has platform-specific
serialization for `PathBuf`:

```rust
impl Decodable for path::PathBuf {
    #[cfg(target_os = "redox")]
    fn decode<D: Decoder>(d: &mut D) -> Result<path::PathBuf, D::Error> {
        // ...
    }
    #[cfg(unix)]
    fn decode<D: Decoder>(d: &mut D) -> Result<path::PathBuf, D::Error> {
        // ...
    }
    #[cfg(windows)]
    fn decode<D: Decoder>(d: &mut D) -> Result<path::PathBuf, D::Error> {
        // ...
    }
}
```

Interestingly enough, this is something I've touched on in a previous post as a [portability concern].

[portability concern]: https://udoprog.github.io/rust/2017-11-05/portability-concerns-with-path.html

In `rustc-serialize`, paths are not portable because not all platforms have serializers defined for them!
`impl`s being missing for a specific platform wouldn't be a big deal if platform-specific bits were better tucked away.
As it stands here, we end up with an incomplete `impl Decodable` which is a compiler error.

If the entire `impl` block was conditional it would simply be overlooked as another missing
implementation. In practice this would mean that you wouldn't be able to serialize `PathBuf` easily, but this can be worked around be simply not using it.

Due to the current state of affairs, the easiest way to deal with it was simply to disable the default features for the `num-*` crates.
A bit tedious to add everywhere, but not a big deal:

```
[package]
# ...

[dependencies]
num-bigint = {version = "0.1", default_features = false}
num-traits = {version = "0.1", default_features = false}
num-integer = {version = "0.1", default_features = false}
```

## There is no filesystem

I lie, [there kind of is]. Or at least there can be.
But for our current target `wasm32-unknown-unknown` there isn't.

[there kind of is]: https://kripken.github.io/emscripten-site/docs/porting/files/file_systems_overview.html

This means that all of your sweet code using `std::fs` simply won't work.

```
//! src/main.rs

#[macro_use]
extern crate stdweb;

use std::fs;

fn test() -> String {
    fs::File::create("hello.txt").expect("bad file");
    "Hello".to_string()
}

fn main() {
    stdweb::initialize();

    js! {
        Module.exports.test = @{test};
    }
}
```

Taking this for a spin, would result in an unfortunate runtime error:

![wasm + fs]({{ "/assets/2018-02-19-wasm-fs.png" | absolute_url }})

To work around this I introduced a layer of indirection.
My very own [`fs.rs`].
I then ported all code to use this so that I can swap out the implementation at runtime.
This wasn't particularly hard, seeing as I already pass around a [`Context`] to collect errors.
Now it just needed to [learn a new trick].

[`fs.rs`]: https://github.com/reproto/reproto/blob/0356c90c60/lib/core/src/fs.rs
[`Context`]: https://github.com/reproto/reproto/blob/0356c90c60/lib/core/src/context.rs
[learn a new trick]: https://github.com/reproto/reproto/blob/0356c90c60/lib/core/src/context.rs#L25

Finally I ported all code that used [`Path`] to use [`relative-path`] instead.
This guarantees that I won't be tempted to hit any of those platform-specific APIs like [`canonicalize`], which requires access to a filesystem.

[`Path`]: https://doc.rust-lang.org/std/path/struct.Path.html
[`relative-path`]: https://crates.io/crates/relative-path
[`canonicalize`]: https://doc.rust-lang.org/std/path/struct.Path.html#method.canonicalize

With this in place I can now [capture the files] written to my filesystem abstraction directly into memory!

[capture the files]: https://github.com/reproto/reproto/blob/0356c90c60/lib/compile/lib.rs#L34

## Ruining Christmas with native libraries

Anything involving native libraries will ruin your day in one way or another.

My [`repository`] component uses `ring` for calculating [Sha256 checksums].
The first realization is that repositories won't work the same - if at all - on the web.
We don't have a filesystem!
At some point it might be possible to plug in a solution that communicates with a service to fetch dependencies.
But that is currently not the goal.

[`repository`]: https://github.com/reproto/reproto/blob/0356c90c60/doc/usage.md#setting-up-and-using-a-repository
[Sha256 checksums]: https://github.com/reproto/reproto/blob/0356c90c60/lib/repository/src/sha256.rs

This realization made the solution obvious: web users don't need a repository.
I moved the necessary trait ([`Resolver`]) from `repository` to `core`, and provided a [no-op] implementation for it.
The result is that I no longer depend on the `repository` crate to have a working system, sidestepping the native dependency entirely in the browser.

Neat!

[`Resolver`]: https://github.com/reproto/reproto/blob/0356c90c60/lib/core/src/resolver.rs

[no-op]: https://github.com/reproto/reproto/blob/0356c90c60/lib/core/src/resolver.rs#L54

## Revenge of the Path

I thought I had seen the last of `Path`. But [`url`] decided to blow up in my face like this:

[`url`]: https://crates.io/crates/url

```
error[E0425]: cannot find function `path_to_file_url_segments` in this scope
    --> <registry>/src/github.com-1ecc6299db9ec823/url-1.6.0/src/lib.rs:1934:32
     |
1934 |         let (host_end, host) = path_to_file_url_segments(path.as_ref(), &mut serialization)?;
     |                                ^^^^^^^^^^^^^^^^^^^^^^^^^ did you mean `path_to_file_url_segments_windows`?
error[E0425]: cannot find function `file_url_segments_to_pathbuf` in this scope
    --> <registry>/src/github.com-1ecc6299db9ec823/url-1.6.0/src/lib.rs:2057:20
     |
2057 |             return file_url_segments_to_pathbuf(host, segments);
     |                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ did you mean `file_url_segments_to_pathbuf_windows`?
error: aborting due to 2 previous errors
error: Could not compile `url`.
```

Yet again. Paths _aren't_ portable.

The `url` crate wants to translate file segments into _real_ paths for you.
It does this by hiding implementations of `file_url_segments_to_pathbuf` behind platform-specific gates.
Obviously there there are no `wasm` implementations for this.

An alternative here is to use something like [`hyper::Uri`], but that would currently mean pulling in all of hyper and its problematic dependencies.
I settled for just adding more indirection and isolating the components that needed HTTP and URLs into their own modules.

[`hyper::Uri`]: https://docs.rs/hyper/0.11.18/hyper/struct.Uri.html

<blockquote>
"All problems in computer science can be solved by another level of indirection"
&mdash; David Wheeler
</blockquote>

## What's the time again?

[`chrono`] is another amazing library, I used it in my [`derive`] component to detect when a `string` looks like a `datetime`.

Unfortunate for me, [`chrono` depends on `time`].
Another platform-specific dependency!

[`derive`]: https://github.com/reproto/reproto/blob/0356c90c60/lib/derive/
[`chrono`]: https://crates.io/crates/chrono
[`chrono` depends on `time`]: https://github.com/chronotope/chrono/blob/master/Cargo.toml#L23

```
error[E0432]: unresolved import `self::inner`
 --> <registry>/src/github.com-1ecc6299db9ec823/time-0.1.39/src/sys.rs:3:15
  |
3 | pub use self::inner::*;
  |               ^^^^^ Could not find `inner` in `self`
error[E0433]: failed to resolve. Could not find `SteadyTime` in `sys`
   --> <registry>/src/github.com-1ecc6299db9ec823/time-0.1.39/src/lib.rs:247:25
    |

/// snip...

error: aborting due to 10 previous errors
error: Could not compile `time`.
```

Because the _derive_ feature was such a central component in what I wanted to port I started looking for alternatives instead of isolating it.

My first attempt was the [`iso8601`] crate, a project using [`nom`] to parse ISO 8601 timestamps.
Perfect!

[`iso8601`]: https://crates.io/crates/iso8601
[`nom`]: https://crates.io/crates/nom

```
error[E0432]: unresolved import `libc::c_void`
  --> <registry>/src/github.com-1ecc6299db9ec823/memchr-1.0.2/src/lib.rs:19:5
   |
19 | use libc::c_void;
   |     ^^^^^^^^^^^^ no `c_void` in the root
error[E0432]: unresolved import `libc::c_int`
  --> <registry>/src/github.com-1ecc6299db9ec823/memchr-1.0.2/src/lib.rs:21:12
   |
/// ...

error: build failed
```

On no...

Ok, it's time to pull out `cargo-tree`.

```bash
$ cargo tree

# ...
├── iso8601 v0.2.0
│   [dependencies]
│   └── nom v3.2.1
│       [dependencies]
│       └── memchr v1.0.2
│           [dependencies]
│           └── libc v0.2.36
# ...
```

So [`nom`] depends on [`memchr`], an interface to the [`memchr` libc function].
That makes sense. `nom` wants to scan for characters as quickly as possible.
Unfortunately that makes `nom` and everything depending on it unusable in `wasm` right now.

[`memchr`]: https://crates.io/crates/memchr
[`memchr` libc function]: https://linux.die.net/man/3/memchr

[`derive`]: https://github.com/reproto/reproto/blob/master/lib/derive/

The easiest route ended up being to [write my own function] here.

[write my own function]: https://github.com/reproto/reproto/blob/master/lib/derive/utils.rs

## Make things better

In the following sections I try to summarize how we can improve the experience for future porters.

### Make platform detection a _first class_ feature of Rust

If you look over the error messages encountered above, you can see that they have one thing in common:
The are all _unique_.

This is unfortunate, since they all relate to the same problem: there is a component `X` that is hidden behind a platform gate.
When that component is no longer provided, the project _fails to compile_.

Wouldn't it be better if the compiler error looked like this:

```
error[EXXXX]: section unavailable for platform (target_arch = "wasm32", target_os = "unknown"):
  --> <registry>/src/github.com-1ecc6299db9ec823/rustc-serialize-X.X.X/src/serialize.rs:148:1
    |
148 |         #[platform(target_arch, target_os)] {
    |                    ^^^^^^^^^^^^^^^^^^^^^^ - platform defined here
    |             impl Decodable for path::PathBuf {
    |                 #[cfg(target_os = "redox")]
    |                 fn decode<D: Decoder>(d: &mut D) -> Result<path::PathBuf, D::Error> { .. }
    |
    |                 #[cfg(target_os = "unix")]
    |                 fn decode<D: Decoder>(d: &mut D) -> Result<path::PathBuf, D::Error> { .. }
    |
    |                 #[cfg(target_os = "windows")]
    |                 fn decode<D: Decoder>(d: &mut D) -> Result<path::PathBuf, D::Error> { .. }
    |             }
    |         }
}
```

It works by detecting when your platform has a configuration which does not match any existing gates, providing you with contextual information of why it failed. This means that the matching would either have to be exhaustive (e.g. provide a default fallback), or fail where the matching actually occurs.

This is much better than the arbitrary number of compiler errors caused by missing elements.

### Transitive feature flags

This is the first exercise I've had in explicitly disabling default features.
Sometimes it can be hard.
Dependency topologies are complex, and mostly out of your hand.

This suggestion is heavily influenced by [`use` flags in Gentoo], and would be a controversial change.

[`use` flags in Gentoo]: https://www.gentoo.org/support/use-flags/

The gist here is that I can enable a feature for a crate and all it's dependencies.
Not just directly on that crate.
This way it doesn't matter that a middle layer forgot to forward a feature flag, you can still disable it without too much hassle.

Different crates might use the same feature flag for different things.
But it begs the question: are we more interested in patching all crates to forward feature flags correctly, than we are patching crates which use the wrong flags?

## Conclusion

This was a lot of work. But _much_ less than I initially expected.
The `wasm` ecosystem in Rust is really on fire right now, and things are improving rapidly!

I'm actually really excited over how well it works.
Apart from a small number of expected, and even smaller number of unexpected dependencies.
Things _just work_.

In summary:

* Avoid native dependencies.
* When something breaks, it's probably because of a gate.
* Abstract the filesystem.
* Avoid using `Path`/`PathBuf` and APIs which have platform-specific behaviors. Consider
  [`relative-path`] as an alternative.

[`relative-path`]: https://crates.io/crates/relative-path

So to leave you, feel free to try out [`reproto`] in your browser using the new [`eval`] app:

<https://reproto.github.io>.

**UPDATE #1:**
Here is the relevant PR in `chrono` to [make `time` an optional dependency].
Please make your voice heard if this is something you want!
For `nom`, memchr support for `wasm` [was merged in December], unfortunately that leaves existing libraries behind until they are upgraded.

[make `time` an optional dependency]: https://github.com/chronotope/chrono/pull/137
[was merged in December]: https://github.com/Geal/nom/issues/633

[`reproto`]: https://github.com/reproto
[`eval`]: https://github.com/reproto/reproto/tree/master/eval

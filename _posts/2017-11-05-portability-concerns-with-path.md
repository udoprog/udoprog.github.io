---
layout: post
title: "Portability concerns with Path"
description: ""
category: rust
---

I've been spending most of my spare time working on [ReProto], and I'm at a point where I need to
support specifying a per-project build manifest.

In this manifest I want to give the user the ability to specify build paths.
The problem I faced is: How do you have a path specification that is portable?

<!-- more -->

The build manifest will be checked into git repositories.
It will shared in verbatim across platforms, and users would expect it to work without having to
convert any paths specified in it to their native representation.
This is very similar to how a build configuration is provided to `cargo` through `Cargo.toml`.
It would _really_ suck if you'd have to convert all back-slashes to forward-slashes, just because
the original author of a library is working on Windows.

Rust has excellent serialization support in the form of [serde].
The following is an example of how you can use serde to deserialize TOML whose structure is
determined by a `struct`.

```rust
extern crate toml;
#[macro_use]
extern crate serde_derive;

use std::path::PathBuf;

#[derive(Debug, Deserialize)]
pub struct Manifest {
    paths: Vec<PathBuf>,
}

const FILE: &'static str = "paths = ['extra', 'src/main/reproto']";

pub fn main() {
    let manifest: Manifest = toml::from_str(FILE).unwrap();
    println!("{:?}", manifest);
}
```

We've deserialized a list of `paths` so our work _seems_ like it's mostly done.

In the next section I will describe some details around platform-specific behaviors in Rust, and
how they come back to bite us in this case.

# Platform behaviors

Representing filesystem paths in a platform-neutral way is an interesting problem.

Rust has defined a platform-agnostic [`Path`] type which has system-specific behaviors implemented
in libstd.
For example, [in Windows] it deals with a prefix consisting of the drive letter (e.g. `c:`).

The effect for our manifest is that using [`PathBuf`] would permit our application to _accept_ and
_operate over_ paths specified in different ways.
The exact of which depends on which platform your application is built for.

This is no good for configuration files that you'd expect people to share across platforms.
One representation might be valid on one platform, but not on others.

The following snippet exemplifies the problem:

```rust
extern crate toml;
#[macro_use]
extern crate serde_derive;

use std::path::{PathBuf, Path};

#[derive(Debug, Deserialize)]
pub struct Manifest {
    paths: Vec<PathBuf>,
}

const FILE: &'static str = "paths = ['foo\\bar']";

pub fn main() {
    let manifest: Manifest = toml::from_str(FILE).unwrap();

    if let Some(path) = manifest.paths.iter().next() {
        let p = Path::new(".").join(path).join("baz");

        println!("path = {:?}", p);
        println!("components = {:?}", p.components().collect::<Vec<_>>());
    }
}
```

On Windows, it would give this output:

```text
path = "./foo\\bar/baz"
components = [CurDir, Normal("foo"), Normal("bar"), Normal("baz")]
```

While on Linux, it would behave differently with:

```text
path = "./foo\\bar/baz"
components = [CurDir, Normal("foo\\bar"), Normal("baz")]
```

`foo\\bar` is treated like a path component, because backslash (`\`) is _not_ a directory separator
on Linux.
The implementation of [`Path`] on Linux reflects this.

This means that mutator functions in Rust will treat this as a component when determining things
like what the parent directory of a given path is:

```rust
use std::path::Path;

pub fn main() {
    let path = Path::new("root").join("foo\\bar");
    let parent = path.parent();
    println!("parent = {:?}", parent);
}
```

On Windows:

```text
parent = Some("root\foo")
```

On Linux:

```text
parent = Some("root")
```

# Portable paths

[`Path`] by itself provides a portable API.
[`PathBuf::push`] and [`Path::join`] are ways to manipulate a path on a per-component basis.
The components themselves might have restrictions on which
[character sets may be used][windows-paths], but at least the path separator can be abstracted away.

Another major difference is how filesystem roots are designated.
Windows, interestingly enough, have multiple roots - one for each drive.
Linux only has one: `/`.

With this in mind we can write portable code that only manipulates _relative_ paths.
These works independently of which platform it is running on:

```rust
use std::path::Path;
use std::env;

fn main() {
    let base = env::current_dir().unwrap();
    let target = base.join("foo").join("bar");
    println!("target = {:?}", target);
}
```

On Windows this gives:

```text
target = "C:\\Users\\udoprog\\foo\\bar" 
```

And on Linux:

```text
target = "/home/udoprog/foo/bar"
```

Notice that the relative `foo/bar` traversal is maintained.

The realization I had is that you can have a portable description if you can describe a path only
in terms of its components, without filesystem roots.

Neither `c:\foo\bar\baz` nor `/foo/bar/baz` are a portable descriptions, `foo/bar/baz` _is_.
It simply states; please traverse `foo`, then `bar`, then `baz`, relative to _some_ directory.

Combining this _relative_ path with a native path allows it to be translated into a
platform-specific path.
This path can then be used for filesystem operations.

This is the premise behind a new crate I created named [`relative-path`], which I will be covering
briefly next.

# Relative paths and the real world

In the [`relative-path`] crate I've introduces two classes: [`RelativePath`] and [`RelativePathBuf`].
These are analogous to the libstd classes [`Path`] and [`PathBuf`].
A fairly significant chunk of code could be reimplemented based on these classes.

The differences from their libstd siblings are small, but significant:

 * The path separator is set to a fixed character (`/`), regardless of platform.
 * Relative paths _cannot_ represent an absolute path in the filesystem, without first specifying
   what they are relative to through [`to_path`].

The second rule is important to either determine the _actual_ relativeness of a Path, or which
filesystem root or drive it belongs to.

This permits using [`RelativePathBuf`] in cases where having a portable representation would
otherwise cause problems across platforms.
Like with [build manifests] checked into a git repository:

```rust
extern crate toml;
#[macro_use]
extern crate serde_derive;
extern crate relative_path;

use relative_path::RelativePathBuf;
use std::path::{PathBuf, Path};

#[derive(Debug, Deserialize)]
pub struct Manifest {
    paths: Vec<RelativePathBuf>,
}

const FILE: &'static str = "paths = ['foo/bar']";

pub fn main() {
    let manifest: Manifest = toml::from_str(FILE).unwrap();

    if let Some(path) = manifest.paths.iter().next() {
        let p = path.to_path(Path::new(".")).join("baz");

        println!("path = {:?}", p);
        println!("components = {:?}", p.components().collect::<Vec<_>>());
    }
}
```

My hope is that you from now on folks won't be relegated to storing stringly typed fields and
is forced to [figure][bug-1] [out][bug-2] the [portability][bug-3] [puzzle][bug-4] for themselves.

# Final notes

Character restrictions are _still_ a problem.
At some point we might want to incorporate replacement procedures, or APIs that return `Result`'s
to flag for non-portable characters.

Using a well-defined path separator gets us pretty far regardless.

Thank you for reading this. And please give me feedback on [`relative-path`] if you have the time.

[Comments on Reddit.][reddit]

[ReProto]: https://github.com/reproto
[in Windows]: https://github.com/rust-lang/rust/blob/master/src/libstd/sys/windows/path.rs
[serde]: https://serde.rs
[`Path`]: https://doc.rust-lang.org/std/path/struct.Path.html
[`PathBuf`]: https://doc.rust-lang.org/std/path/struct.PathBuf.html
[`PathBuf::push`]: https://doc.rust-lang.org/std/path/struct.PathBuf.html#method.push
[`Path::join`]: https://doc.rust-lang.org/std/path/struct.Path.html#method.join
[`RelativePath`]: https://docs.rs/relative-path/0.1.5/relative_path/struct.RelativePath.html
[`RelativePathBuf`]: https://docs.rs/relative-path/0.1.5/relative_path/struct.RelativePathBuf.html
[windows-paths]: https://msdn.microsoft.com/en-us/library/windows/desktop/aa365247(v=vs.85).aspx
[`to_path`]: https://docs.rs/relative-path/0.1.5/relative_path/struct.RelativePath.html#method.to_path
[`relative-path`]: https://crates.io/crates/relative-path
[build manifests]: https://github.com/udoprog/reproto/commit/0c1c1ea4fb8e12919e0c420a174ea67e5da6ffcc
[bug-1]: https://github.com/rust-lang/rust-installer/issues/65
[bug-2]: https://github.com/racer-rust/racer/issues/47
[bug-3]: https://github.com/rust-lang/rfcs/issues/1092
[bug-4]: https://github.com/alexcrichton/cargo-vendor/issues/22
[reddit]: https://www.reddit.com/r/rust/comments/7b22im/portability_concerns_with_path/

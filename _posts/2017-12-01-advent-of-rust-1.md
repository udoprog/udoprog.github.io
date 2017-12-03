---
layout: post
title: "Advent of Rust Day 1 - Everything has a scope"
description: ""
category: rust
---

This is my new series where I'll be discussing interesting Rust tidbits that I encountered when
solving [Advent of Code 2017].

You can find the complete (spoiler) solution here: [udoprog/rust-advent-of-code-2017]

[Advent of Code 2017]: http://adventofcode.com/2017
[udoprog/rust-advent-of-code-2017]: https://github.com/udoprog/rust-advent-of-code-2017

<!-- more -->

I decided the use the following compact expression to open and read the entire contents of a file:

```rust
let mut data = String::new();
File::open(path)?.read_to_string(&mut data)?;
```

Desugared, it would read as something like this:

```rust
let mut data = String::new();

{
    let mut _f = File::open(path)?;
    _f.read_to_string(&mut data)?;
}
```

This means that the file is closed immediately after its contents has been read, and is not
accessible anywhere else in the scope.

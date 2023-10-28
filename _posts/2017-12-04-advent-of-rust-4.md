---
layout: post
title: "Advent of Rust Day 4 - Testing things that read stuff"
description: ""
category: rust
---

This is a series where I'll be discussing interesting Rust tidbits that I encounter when solving
[Advent of Code 2017].

You can find the complete (spoiler) solution here: [udoprog/rust-advent-of-code-2017]

[Advent of Code 2017]: http://adventofcode.com/2017
[udoprog/rust-advent-of-code-2017]: https://github.com/udoprog/rust-advent-of-code-2017

<!-- more -->

If you want to test a function that reads stuff, you can do the following:

```rust
use std::io::Read;

fn my_fun<R: Read>(reader: R) {
    /*  */
}
```

This allows you to write tests like these:

```rust
#[test]
fn test_with_cursor() {
    use std::io::Cursor;
    my_fun(Cursor::new("hello\nworld"));
}
```

If you don't like monomorphization (large binaries?), use a trait object instead:

```rust
use std::io::Read;

fn my_fun(reader: &mut Read) {
    /*  */
}
```

But pay the price of dynamic dispatch.

---
layout: post
title: "Advent of Rust Day 10 - Persistent iterators"
description: ""
category: rust
---

This is a series where I'll be discussing interesting Rust tidbits that I encounter when solving
[Advent of Code 2017].

You can find the complete (spoiler) solution here: [udoprog/rust-advent-of-code-2017]

[Advent of Code 2017]: http://adventofcode.com/2017
[udoprog/rust-advent-of-code-2017]: https://github.com/udoprog/rust-advent-of-code-2017

<!-- more -->

I've realized that a powerful alternative to doing (often unsafe!) arithmetics and indexing is to
make use of iterators.

One pattern I've found myself moving to is reusing a mutable iterator, like this:

```rust
// store an iterator
let mut skip = 0..;

for i in 1..4 {
    for (n, skip) in (0..i).zip(&mut skip) {
        println!("n = {}, skip = {}", n, skip);
    }
}
```

[Playground Link](https://play.rust-lang.org/?gist=3fb936c56aa474cc80b98bcf006824a8&version=stable)

This would print:

```
n = 0, skip = 0
n = 0, skip = 1
n = 1, skip = 2
n = 0, skip = 3
n = 1, skip = 4
n = 2, skip = 5
```

Notice how the `skip` iterator is shared across all iterations.

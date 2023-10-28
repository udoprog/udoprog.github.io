---
layout: post
title: "Advent of Rust Day 6 - The power of iterators"
description: ""
category: rust
---

This is a series where I'll be discussing interesting Rust tidbits that I encounter when solving
[Advent of Code 2017].

You can find the complete (spoiler) solution here: [udoprog/rust-advent-of-code-2017]

[Advent of Code 2017]: http://adventofcode.com/2017
[udoprog/rust-advent-of-code-2017]: https://github.com/udoprog/rust-advent-of-code-2017

<!-- more -->

On day 6 we need to iterate over a vector, by starting at an index in the middle of the vector and
wrap around.

A neat way to accomplish this, is to use range expressions (like `0..10`) as an iterator, and split
it into two parts.

```rust
let mut v = vec![0, 0, 0, 0];
let idx = 2;

for (i, idx) in (idx..v.len()).chain(0usize..idx).enumerate() {
    v[idx] += i;
}

println!("out = {:?}", v);
```

This would print:

```
out = [2, 3, 0, 1]
```

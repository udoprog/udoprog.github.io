---
layout: post
title: "Advent of Rust Day 5 - Have your bounds and eat them too!"
description: ""
category: rust
---

This is a series where I'll be discussing interesting Rust tidbits that I encounter when solving
[Advent of Code 2017].

You can find the complete (spoiler) solution here: [udoprog/rust-advent-of-code-2017]

[Advent of Code 2017]: http://adventofcode.com/2017
[udoprog/rust-advent-of-code-2017]: https://github.com/udoprog/rust-advent-of-code-2017

<!-- more -->

Attempting to access data that is out of bounds in a collections could potentially ruin your day.

To combat this, rust provides 'safe' alternatives for slice indexing in the form of [`get`] and
[`get_mut`].

[`get`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.get
[`get_mut`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.get_mut

*Note*: these are available on `Vec` because it implements `Deref<Target = [T]>`, which causes rust
to look for methods there as well.

These return an `Option<&T>` and `Option<&mut T>`, requiring you to check that the provided index
was present, before attempting to deal with the data.

So this:

```rust
let mut v = vec![1, 2, 2];
println!("data = {}", v[2]);
```

Becomes this:

```rust
let mut v = vec![1, 2, 2];

match v.get(2) {
  Some(value) => println!("data = {}", value),
  None => println!("no value present :("),
}
```

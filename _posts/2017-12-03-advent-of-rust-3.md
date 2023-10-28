---
layout: post
title: "Advent of Rust Day 3 - Option as an Iterator"
description: ""
category: rust
---

This is a series where I'll be discussing interesting Rust tidbits that I encounter when solving
[Advent of Code 2017].

You can find the complete (spoiler) solution here: [udoprog/rust-advent-of-code-2017]

[Advent of Code 2017]: http://adventofcode.com/2017
[udoprog/rust-advent-of-code-2017]: https://github.com/udoprog/rust-advent-of-code-2017

<!-- more -->

Option can be used as an Iterator:

```rust
fn cell_value(storage: &HashMap<(i64, i64), u64>, x: i64, y: i64) -> u64 {
    [
        &(x - 1, y - 1), &(x  , y - 1), &(x + 1, y - 1),
        &(x - 1, y    ), /* (x, y)   */ &(x + 1, y    ),
        &(x - 1, y + 1), &(x  , y + 1), &(x + 1, y + 1),
    ].into_iter().flat_map(|k| storage.get(k)).map(|v| *v).sum()
}
```

Note that `storage.get(k)` returns an `Option`, And the `flat_map` closure returns `U:
IntoIterator`.

`Option` implements `IntoIterator`, which for `None` is an empty iterator, and `Some(T)` is an
iterator with a single item. You can see that in action for the `Item` struct which is used to
implement this behavior: <https://doc.rust-lang.org/src/core/option.rs.html#912>

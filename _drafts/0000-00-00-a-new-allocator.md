---
layout: post
title: A new allocator for Müsli
date: 0000-00-00 00:00 +0200
category: rust
tags: musli
---

You usually don't see the allocator in Rust, but typical Rust uses it all the
time.

Types such as `String` are by default backed by an allocation provided through
the [System] allocator. There is an effort to provide an allocator API to make
this behavior modular, but it's slow going. Understandable given how fundamental
such an API is.

<!-- more -->

As I was writing Müsli, I realised that some formats need to "allocate". The
following two examples come to mind:

* When encoding something using `#[musli(pack)]`, Müsli runs a particular pack
  encoder which just serializes one line after another. If the length of the
  packed value cannot be statically known such as when it contains a collection,
  all you can really do is encode it and then get the length of how much you've
  encoded. You need a buffer to do this, and buffers need to be allocated.
* When decoding strings in JSON, it is very likely that we have to make use of a
  ["scratch buffer"]. JSON strings can contain escape sequences, which means
  that it's less likely that the literal string in JSON matches the encoding
  that you're requesting.

[System]: https://doc.rust-lang.org/std/alloc/struct.System.html
["scratch buffer"]: https://github.com/udoprog/musli/blob/aed727771817910c70c14fcf4e5a12b487a84bbc/crates/musli-json/src/reader/string.rs#L54

Each allocation is owned locally so that it can hold metadata beyond what's
available through a pointer.

Owning allocations like that means that neat tricks can be employed, like
efficiently merging adjacent regions.

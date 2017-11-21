---
layout: post
title: "Building services with reproto"
description: ""
category: rust
---

APIs are ubiquitous.

The most popular form by far are JSON-based HTTP APIs (all though GraphQL are giving them a run for
their money). Sometimes these are referred to as restful - because we collectively have an aversion
towards taking REST seriously.

This post isn't about REST.
It's about a project I've been working on for the last year to handle the lifecycle of JSON-based
APIs:

[Rethinking Protocols - reproto](https://github.com/reproto/reproto).

<!-- more -->

reproto is a number of things, but most importantly it's an interface description language (IDL) in
which you can write specifications that describe the structure of JSON objects.
This IDL aims to be compact and descriptive.

A simple `.reproto` specification looks like this:

```
# File: src/cats.reproto

type Cat {
  name: string;
}
```

This describes an object which has a single field `name`, like: `{"name": "Charlie"}`.

Using reproto, we can now generate bindings for this in various languages.

```
$ reproto build --lang rust --package cats --path src --out src/generated
```

For Rust, this would be using [Serde]:

[Serde]: https://serde.rs

```rust
// File: src/generated/cats.rs

#[derive(Serialize, Deserialize, Debug)]
struct Cat {
  name: String,
}
```

In Java, [Jackson] would be used:

[Jackson]: https://github.com/FasterXML/jackson

```java
// File: src/main/java/Cat.java

import lombok.Data;

@Data
public static class Cat {
  private final String name;

  @JsonCreator
  public Cat(@JsonProperty("name") final String name) {
    this.name = name;
  }
}
```

reproto tries to integrate with the target language using the best frameworks available[^1].

[^1]: The exact approach is configurable through modules documented under [Language Support](https://github.com/reproto/reproto/blob/master/doc/spec.md#language-support).

# Dependencies

A system is something greater than the sum of its parts.

Say you want to write a service that communicate with with many other services, it's typically
painful and error prone to copy things around by yourself.

To solve this reproto is not only a language specification, but also a package manager.

Provide reproto with a build manifest in `reproto.toml` like this:

```toml
language = "rust"
output = "src/generated"

[modules.chrono]

[packages]
"io.reproto.toystore" = "^1"
```

Run:

```
$ reproto update
$ reproto build
```

And reproto will have downloaded and built `io.reproto.toystore` from the [central repository].

[central repository]: https://github.com/reproto/reproto-index

Importing a manifest from somewhere else inside of a specification will automatically use the
repository:

```
use io.reproto.toystore "^1" as toystore;

type Shelf {
  toys: [toystore::Toy];
}
```

Dealing with many different versions of a package is handled through clever namespacing.

This makes it possible to import and use multiple different versions of a specification at once:

```
use io.reproto.toystore "^1" as toystore1;
use io.reproto.toystore "^2" as toystore2;

type Shelf {
  toys: [toystore1::Toy];
  toys_v2: [toystore2::Toy];
}
```

# Documentation

Good documentation is key to effectively using an API.

reproto comes with a built-in documentation tool in `reproto doc`, which will generate
documentation for you by reading rust-style [documentation comments].

You can check out the example [documentation for `io.reproto.toystore` here][doc-example].

[documentation comments]: https://github.com/reproto/reproto/blob/master/doc/spec.md#documentation
[doc-example]: https://reproto.github.io/reproto/doc-examples/io/reproto/toystore/1.0.0/index.html

# Fearless versioning

With package management comes the problems associated with _breaking changes_.

reproto insists on using semantic versioning, and will actively check that any version you try to
publish doesn't violate it:

```bash
$ reproto publish
src/io/reproto/toystore.reproto:12:3-22:
 12:   category: Category;
       ^^^^^^^^^^^^^^^^^^^ - minor change violation: field changed to be required
io.reproto.toystore-1.0.0:12:3-23:
 12:   category?: Category;
       ^^^^^^^^^^^^^^^^^^^^ - from here
```

This is all based on a [module named `semck`][semck] that operates on the AST-level.

Not everything is covered yet, but it's rapidly getting there.

[semck]: https://github.com/reproto/reproto/tree/master/semck

# Finally

In contrast to something like purely an api specification language, reproto aims to be a complete
system to hold your hands during the entire lifecycle of service development.

My litmus test will be when I've produced a mostly generated client for [Heroic], which is [well on
its way][heroic-api].

It's also written in [Rust], a language where a lot of these ideas have been shamelessly stolen
from.

There is still a lot of work to be done!
If you are interested in the problem domain and have spare cycles, please join me on [Gitter].

[Comments on reddit][reddit].

[Rust]: https://www.rust-lang.org/
[spec]: https://github.com/reproto/reproto/blob/master/doc/spec.md
[Heroic]: https://github.com/spotify/heroic
[heroic-api]: https://github.com/udoprog/heroic/commit/api
[Gitter]: https://gitter.im/reproto/reproto
[reddit]: https://www.reddit.com/r/rust/comments/7ehe3l/building_services_with_reproto/

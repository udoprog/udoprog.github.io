---
layout: post
title: "Let's talk about JSON (1)"
description: ""
category: programming
---

Over the last year I've spent a lot of time working with JSON.
I suspect most of you reading this have as well.

Whether we like it or not, JSON is currently the lowest common denominator for how we convince
machines to talk to each other.

In this blog series I will be discussing how we _work_ with JSON.
The good, and the bad. Mostly the bad honestly, since those make for a more interesting read.
But I'm also hoping to start a conversation about good practices in how we design schemas to
maximize interoperability.

<!-- more -->

I will frequently be referencing the work I do in relation to [reproto], since that is what
brought me down this route in the first place.
reproto is a system for describing the schema of JSON, so that the valid documents they represent
can be used efficiently and safely across languages and frameworks.

First I'd like to examine soundness in relation to JSON.

## What is soundness?

JSON is a self-descriptive format.
It includes notation for how to describe objects, arrays, and values.
This alone is not sufficient to build programs which safely deal with the data it processes.
We need to couple it with _structure_, the way we expect it to be put together.

A _schema_.

So this is how I think about sound JSON:

1. It's Unambiguous &mdash; for any given structure there is one clear way to interpret what it
   means.
2. It can be processed in an idiomatic way by a majority of systems dealing with JSON.

A sound approach would be to use a discriminator field (e.g. `type`) to determine which type of
structure to expect:

```json
{"type": "eagle", "age": 15}
{"type": "butterfly", "color": "red"}
```

An unsound approach would be to determine it based on the presence of fields. If `age` is present,
it's an eagle. If `color` is present, it's a butterfly.

```json
{"age": 15}
{"color": "red"}
```

The first approach also fulfills the second.
It's called field-based polymorphism and is widely supported in one form or another.

But I don't use schemas, what does this have to do with me?

## The implicit schema

Schemas are _always_ present.  Even the following super-simple Python application assumes a
schema:

```python
import sys, json
d = json.load(sys.stdin)
print("My name is: " + d["name"])
```

Running this will yield:

```bash
python3 app.py <<< '{"name": "George"}'
My name is: George
```

But what happens if we feed it a document without a `name`?

```bash
python3 app.py <<< '{}'
Traceback (most recent call last):
  File "app.py", line 3, in <module>
    print("My name is: " + d["name"])
KeyError: 'name'
```

What if we feed it a `name` that is not a string?

```bash
python3 app.py <<< '{"name": 42}'
Traceback (most recent call last):
  File "app.py", line 3, in <module>
    print("My name is: " + d["name"])
TypeError: must be str, not int
```

What if we don't feed it an object at all?

```bash
python3 app.py <<< '"George"'
Traceback (most recent call last):
  File "app.py", line 3, in <module>
    print("My name is: " + d["name"])
TypeError: string indices must be integers
```

We can observe that due to the way the application is written, it has an implicit assumption about
the JSON it accepts as valid.

We could write this schema like this:

```reproto
type Person {
    /// Name of the person.
    name: string;
}
```

But what if the application was written like this instead?

```python
import sys, json
d = json.load(sys.stdin)
print("My name is:", d["name"])
```

Now we actually accept anything as a name now without rejecting the input.

```bash
python3 test.py <<< '{"name": {"first": "George"}}'
My name is: {'first': 'George'}
```

Or expressed as a schema:

```reproto
type Message {
    /// Name of the person.
    /// Can be anything really.
    name: any;
}
```

Here [duck typing][duck-typing] kicks in, Python accepts almost anything as an argument to print,
assuming that intent of the programmer is to "make it a string".

But let's take a step back. Is this sound?

[reproto]: https://reproto.github.io
[duck-typing]: https://en.wikipedia.org/wiki/Duck_typing

## When do we want our code to fail?

Preferably _never_, but if we have to: in the decoding stage.

It's reasonable to design our application so that you can be sure that beyond a given point the
structure is well-defined.

The decoding stage in the above example would be the line:

```python
d = json.load(sys.stdin)
```

By permitting [opaque structures] to fall through the decoding stage, we expose ourselves to
unexpected behaviors.

This is why Python 2 developers adopted the adage "decode early/encode late" in relation to
unicode.
Unless you provide some constraints, your problem space becomes so unconstrained that it's almost
meaningless to reason about.

Let's rewrite our program to fail if we are not receiving well-formed JSON:

```python
import sys, json

def validate(d):
    # check that it's an object
    if not isinstance(d, dict):
        raise Exception("data is not an object")

    # check that name is present
    if "name" not in d:
        raise Exception("name: field is not present")

    name = d["name"]

    # check that name is a string
    if not isinstance(name, str):
        raise Exception("name: field is not a string")

d = json.load(sys.stdin)
validate(d)

print("My name is: " + d["name"])
```

While the above is not particularly pretty, it guarantees that after have called `validate`, `d` is
well-formed and the rest of the code behaves as expected.

In the case we are not receiving the data we expected, we provide at least some diagnostics telling
us why.

If we go one step further, we might wrap our data in a class:

```python
import sys, json

class Person:
    def __init__(self, name):
        self.name = name

    @staticmethod
    def decode(d):
        if not isinstance(d, dict):
            raise Exception("data is not an object")

        if "name" not in d:
            raise Exception("name: field is not present")

        name = d["name"]

        if not isinstance(name, str):
            raise Exception("name: field is not a string")

        return Person(name)

d = Person.decode(json.load(sys.stdin))
print("My name is: " + d.name)
```

Now we can provide stronger guarantees: any instance of Person _must_ be well-formed.
We've also coupled our decoding method with the class as a static method.

As a side-effect, invalid property access is also more well-defined!
If we try to access unknown properties, we would raise an `AttributeError`.

```python
p = Person("John")
print(p.age)
```

Would give:

```
Traceback (most recent call last):
  File "test.py", line 23, in <module>
    print(p.age)
AttributeError: Person instance has no attribute 'age'
```

Our application now enforces a _schema_.
Any data we process here, can be safely interchanged with other systems that follows the same
schema.

[opaque structures]: https://en.wikipedia.org/wiki/Opaque_data_type

## How does this look like in Java?

Java is statically typed.

Dealing with raw JSON in Java is exceedingly unergonomic.
A more common approach is to employ a strategy known as [data binding].

We would define classes that represent our schema, and use a framework like [Jackson] to "bind"
our data to these classes:

```java
public class Main {
    public static class Person {
        private final String name;

        @ConstructorProperties({"name"})
        public Person(final String name) {
            this.name = name;
        }

        public String getName() {
            return this.name;
        }
    }

    public static void main(String[] argv) throws Exception {
        final ObjectMapper m = new ObjectMapper();
        m.setSerializationInclusion(Include.NON_NULL);

        final String json = "{\"name\": \"George\"}";
        final Person person = m.readValue(json, Person.class);
    }
}
```

[Jackson]: https://github.com/FasterXML/jackson
[data binding]: https://en.wikipedia.org/wiki/Data_binding

## Sharing is caring

In order for these two systems to communicate - one in Python, the other in Java - we need a way to
describe our _schema_ that can be implemented in both languages.

Even though we never touched on a schema in any of the previous code, it still has one.

In reproto, a `Person` would be expressed like this:

```reproto
type Person {
    name: string;
}
```

[You can try it out here](https://reproto.github.io).

## Not all solutions are born equal

Figuring out how to only permit legal JSON according to a schema is something I had to do, and is
a recurring problem with the code generation for reproto.

In future posts I will be detailing some of the challenges I encountered with certain languages and
framework combinations.
I'll also be highlighting the ones I believe are excellent, and why.

I will kick off this series, at the end of this post, by stating what I believe soundness is with
relation to JSON:

 1. Any given JSON when paired with a schema can be unambiguously decoded or rejected.
 2. Any given JSON can be processed in a language, without sacrificing guarantees provided _by_
    that language.

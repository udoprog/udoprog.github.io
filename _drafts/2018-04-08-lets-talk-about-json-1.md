---
layout: post
title: "Let's talk about JSON (1)"
description: ""
category: programming
---

Over the last year I've spent a lot of time working with JSON.
I suspect a lot of you reading this have as well.

Designing an API that speaks JSON over HTTP affords us interoperability with a myriad of languages
and environments.
In a sense, it's currently the lowest common denominator for how we convince machines to talk to
each other.

<!-- more -->

In this blog series I will be discussing _how_ we work with JSON.
We pick it as a format to increase interoperability.
But if we're not careful we might end up hurting it instead.

A common mistake we do is that we assume that things are portable because "JSON is portable".
I'll try to show that the way that we _structure_ our data is even more relevant.
To this end I will be discussing a concept called _soundness_.
This can also be known as: how not to make the lives of our API consumers miserable.

## What is soundness?

JSON is a self-descriptive format.
It includes notation for how to describe objects, arrays, and values.
This alone is not sufficient to build programs which safely deal with the data it processes.
We need to couple it with _structure_, the way we expect it to be put together.

A _schema_.

So this is what I believe sound JSON looks like:

1. It is unambiguous &mdash; for any given structure there is one clear way to interpret what it
   means.
2. It can be processed in an idiomatic way by a majority of systems.
3. The structure makes as few compromises as feasible.

An example of a sound approach to accomplish polymorphism would be to use a discriminator field
(e.g. `type`) to discriminate which type of structure to expect:

```json
{"type": "eagle", "age": 15}
{"type": "butterfly", "color": "red"}
```

This is called field-based polymorphism, and is widely supported in one form or another across many
systems.

It is unambiguous, the type is strictly defined by a single, well-known field.
Because it is widely supported, we get idiomatic implementations for most languages.
There is one compromise: the `type` field cannot be used for anything other than type
discrimination.

An _unsound_ approach would be to determine it based on the presence of fields. If `age` is present,
it's an eagle. If `color` is present, it's a butterfly.

```json
{"age": 15}
{"color": "red"}
```

Strictly speaking this is unambigious.
In [Serde] it would be known as an [untagged enum].
This does _not_ result in idiomatic implementations across systems! Serde is one of the few
exceptions I've encountered where this is easy to express.
There are a number of compromises that have been made: eagle's can never have a _color_, and
butterflies an _age_.
This negatively effect future _schema evolution_ &mdash; something I will cover in a future topic.

But I don't use schemas, what does this have to do with me?

[serde]: https://serde.rs
[untagged enum]: https://serde.rs/enum-representations.html#untagged

## The implicit schema

Schemas are always present.
Even the following super-simple Python application assumes a schema:

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

So what happens if we feed it a document without a `name`?

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

As you see, the way the application is written, it has an _implicit_ assumption about the JSON it
accepts as valid.
This is the _implicit schema_.

In [reproto], we can describe this schema like this:

```reproto
type Person {
    /// Name of the person.
    name: string;
}
```

So what if the application was written like this instead?

```python
import sys, json
d = json.load(sys.stdin)
print("My name is:", d["name"])
```

Now we actually accept _anything_ as a name.

```bash
python3 test.py <<< '{"name": {"first": "George"}}'
My name is: {'first': 'George'}
```

Or expressed as a schema:

```reproto
type Message {
    /// Name of the person.
    /// Can be anything.
    name: any;
}
```

Here [duck typing][duck-typing] kicks in, Python accepts almost anything as an argument to print,
assuming that intent of the programmer is to "make it a string".

But let's take a step back. Is this sound?

[duck-typing]: https://en.wikipedia.org/wiki/Duck_typing

## When do we want our code to fail?

Preferably _never_, but if we have to it would be in the decoding stage.

It's reasonable to design our application so that you can be sure that beyond a given point the
structure is well-defined.
The "decoding stage" in the previous example would be this line:

```python
d = json.load(sys.stdin)
```

This permits [opaque structures] to pass through.
Python will represent the decoded JSON as a set of dynamically composed dictionaries, values, and
lists.
At this stage, if we don't actively verify that we get what we expect, we are exposing ourselves to
unexpected behaviors.

This is why Python 2 developers adopted the adage "decode early/encode late" in relation to
unicode.
Unless you provide some guarantees, your problem space becomes so unconstrained that it's
meaningless to reason about.
Much less guarantee that it is correct.

Let's rewrite our program to fail if we are not receiving what we expect to:

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

While the above is not particularly pretty, it guarantees that after we have called `validate`, the
variable `d` is _well-formed_ and the rest of the code should behave as expected.

In the case we are not receiving the data we expect, we try to provide immediate diagnostics
telling us _why_.
`Exception: "name: field is not present"` with a stack trace relevant to the decoding stage is much
better than a `KeyError: 'name'` raised the first time we try to use it.

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

We also get a number of convenient side-effects.
IDEs can inspect the class and provide completions.
Attempting to access an illegal property gives us a helpful `AttributeError`:

```
Traceback (most recent call last):
  File "test.py", line 23, in <module>
    print(p.age)
AttributeError: Person instance has no attribute 'age'
```

This is a result of us taking an idiomatic approach to how we expose our Python application to
Data, model it as a class instead of as a dynamically structured dictionary.
We play along better with the existing ecosystem.

Our application now actively enforces a _schema_, and as a result any data we process here can be
safely interchanged with other systems that follows it.

[opaque structures]: https://en.wikipedia.org/wiki/Opaque_data_type

## How does this look like in Java?

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

This is where [reproto] comes in.
In it we can describe the structure of our JSON in a language-neutral way, and generate
_idiomatic_, safe code for all of our targets.

The `Person` reference above would simply look like this:

```reproto
type Person {
    name: string;
}
```

[You can try it out here](https://reproto.github.io).

## Not all solutions are born equal

In future posts I'll be covering a number design patterns I've encountered over my career and
discuss how _sound_ they are.

This topic came into focus for me because of my work with [reproto].
I'm trying to come up with a better way for defining and sharing JSON schemas.
It's a _really big_ task, and I would love your help!

[reproto]: https://reproto.github.io

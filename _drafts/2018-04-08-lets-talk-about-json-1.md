---
layout: post
title: "Let's talk about JSON (1)"
description: ""
category: programming
---

Over the last year I've spent a lot of time working with JSON.
I suspect most of you reading this have as well.

This is the first part of a blog series in which I will be investigating the good, the bad, and the
terrifying parts about building systems that interact with each other using JSON.

<!-- more -->

I will frequently be referencing the work I do in relation to [reproto], since that is what
motivated me to write this series to begin with.
To this end, I will give a brief primer on what reproto is.

JSON is a self-descriptive format.
It includes notation for how to describe objects, arrays, and values.
While powerful concepts, this is generally not sufficient to build programs which safely deal with
JSON.

Consider the following Python application:

```python
import sys, json
d = json.load(sys.stdin)
print("My name is: " + d["name"])
```

Let's run it:

```bash
python3 app.py <<< '{"name": "George"}'
My name is: George
```

What happens if we feed it a document without a `name`?

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

Python is really forgiving with data structures and types.
It employs a system called ["duck typing"][duck-typing].

Let's back up for a second and ask the following questions:

 * When do we _want_ our application to fail?
 * How does this look in a statically typed language like Java?

[reproto]: https://reproto.github.io

## When do we want our application to fail?

I propose that it would be preferred if our application fails on line #2:

```python
d = json.load(sys.stdin)
```

By permitting [opaque structures] to fall through the decoding stage, we expose ourselves to
unexpected behaviors.

This is why we might use the adage "decode early".
We want our applications to reject bad input as early as possible to provide guarantees.
These guarantees are foundational to writing safe applications.

Let's rewrite our program to fail if we are not receiving well-formed JSON:

```python
import sys, json

def validate(d):
    if not isinstance(d, dict):
        raise Exception("data is not an object")

    if "name" not in d:
        raise Exception("name: field is not present")

    name = d["name"]

    if not isinstance(name, str):
        raise Exception("name: field is not a string")

    if len(name) <= 0:
        raise Exception("name: must be non-empty")

d = json.load(sys.stdin)
validate(d)

print("My name is: " + d["name"])
```

While the above is not particularly pretty, it guarantees that we have called `validate`, `d` is no
longer opaque.

We guarantee that `name` is present, and is a non-empty string.
If this is not the case, we try to provide useful diagnostics as to why the object is not valid.

We can go one step further and wrap our data in a class.

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

We want to make sure that any instance of `Person` is well-formed.

When we encounter this class anywhere else in our application we've constrain how this data can
interact with our application.

This makes our application safer.

[opaque structures]: https://en.wikipedia.org/wiki/Opaque_data_type
[duck-typing]: https://en.wikipedia.org/wiki/Duck_typing

## How does this look in a language like Java?

Different languages have different ideas of what they consider "safe".
I expect that many Pythonistas are fine with dealing with raw JSON.
This approach would straight up _not_ be an option in Java because the approach would be fatally
unergonomic.

In Java we would rather employ [data binding].
We would define classes that represent the structure of our JSON documents, and use a framework
like [Jackson] to "bind" our data to these classes:

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
        m.setSerializationInclusion(Include.NON_ABSENT);

        final String json = "{\"name\": \"George\"}";
        final Person person = m.readValue(json, Person.class);
    }
}
```

Due to the way Java works, Jackson has to guarantee that `name` is present and is a String.

[Jackson]: https://github.com/FasterXML/jackson
[data binding]: https://en.wikipedia.org/wiki/Data_binding

## Sharing is caring

In order for these two systems to communicate - one in Python, the other in Java - and have the
same desirable safety benefits we need a way to describe the structure of permissible JSON values.

This is where Reproto comes in, we've defined a JSON-oriented interface description language that
permits developers to write and share these definitions.

The above structure, would be written in Reproto like this:

```reproto
type Person {
    name: string;
}
```

[You can try it out here](https://reproto.github.io).

Designing an API that uses objects that can be safely represented in many languages is a frequently
recurring problem.

It's important to identify patterns which should be considered _unsound_, because they limit
interoperability.
Interoperability is the central thesis for _why_ we picked JSON to begin with.

## Not all frameworks are born equal

The process of building backends capable of generating these kind of language bindings has been
an interesting journey.

There is such wide range of solutions available.
In future posts I will be discussing my favorite examples of languages and libraries that _fall
a bit short_.

The intent is not to criticize the developers of these solutions, but to open up a discussion about
what soundness means in the context of using JSON as an interchange format.

I will start by stating what I believe soundness is:

 1. Any given JSON when paired with a schema can be unambiguously decoded or rejected.
 2. Any given JSON can be processed in a language, without sacrificing guarantees provided _by_
    that language.

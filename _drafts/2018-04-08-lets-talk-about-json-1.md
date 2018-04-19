---
layout: post
title: "Let's talk about JSON (1)"
description: ""
category: programming
---

Over the last year I've spent a lot of time working with JSON.
I suspect most of you reading this have as well.

This is unsurprising - JSON is currently the lowest common denominator for how we convince machines
to talk to each other.

In this blog series I will be discussing how we _work_ with JSON.
The good, and the bad. Mostly the bad honestly, since those make for a more interesting read.
But I'm also hoping to start a conversation about good practices in how we design data structures
to maximize interoperability.

<!-- more -->

I will frequently be referencing the work I do in relation to [reproto], since that is what
brought me down this route in the first place.
reproto is a system for describing the schema of JSON, so that the legal documents they represent
can be used efficiently and safely across languages and frameworks.

In order to accomplish this, I had to investigate how different languages, libraries, and
frameworks deal with JSON.
And answer: what does soundness mean in relation to JSON?

## What is soundness?

JSON is a self-descriptive format.
It includes notation for how to describe objects, arrays, and values.
These alone are generally not sufficient to build programs which safely deal with the data it
processes.
We generally need to couple it with a _structure_.
Some structures are legal, and some are not. More formally we would call this a _schema_.

Schemas are _always_ present.  Even the following python application has a schema:

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

Our application interacted with the data in different ways, depending on what data we feed it.
Arguably, only one of these ways were in the manner that the original programmer intended.
The others were rather unexpected.

This means that our application has a schema.
We _expect_ our input to have a particular structure so that we can do something meaningful from
it. Otherwise we will reject it.

Let's back up for a second and ask the following questions:

 * When do we _want_ our application to fail?
 * Python being [duck typed][duck-typing] language is quite flexible in what it permits as input,
   but how do we want this to work in a statically typed language like Java?

[reproto]: https://reproto.github.io
[duck-typing]: https://en.wikipedia.org/wiki/Duck_typing

## When do we want our application to fail?

I propose that it would be preferred if our application fails on line #2:

```python
d = json.load(sys.stdin)
```

By permitting [opaque structures] to fall through the decoding stage, we expose ourselves to
unexpected behaviors.

This is why we might use the adage "decode early".
We _want_ our application to reject bad input as early as possible.
This guarantee is fundamental to writing safe applications.
Validating _all input_ in every function is just not viable.
Instead we set checkpoints in our application that: beyond this point, I have guaranteed that my
data is _sound_.

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

In the case we are not receiving the data we expected, we provide some useful diagnostics telling
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

Now we can provide stronger guarantees: any instance of Person _must_ be well-formed*.
We've also coupled our decoding method with the class as a static method.
Nice!

This has a couple of benefits: If we want to dynamically check what something is, and it is
a `Person`, we know that it is well-formed.
Property access is also more well-defined, in that trying to access unknown properties raises an
`AttributeError`.

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

This makes our application safer.

[opaque structures]: https://en.wikipedia.org/wiki/Opaque_data_type

## How does this look like in Java?

Java is statically typed.
We can actually build a class, with a combination of language invariants and a bit of runtime
checking gives us the above guarantees and more.

Dealing with raw JSON in Java is incredibly unergonomic.
A much more common approach is to employ [data binding].

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
        m.setSerializationInclusion(Include.NON_NULL);

        final String json = "{\"name\": \"George\"}";
        final Person person = m.readValue(json, Person.class);
    }
}
```

Jackson now has to guarantee that `name` is present, and is a String.

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

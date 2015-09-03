---
layout: post
title: "Introducing @AutoSerialize"
description: ""
category: programming
tags: [java]
---

In this post I'll present a new framework for serialization in Java that I've
been working on over the last year.

The initial goal was to implement a paradigm which I felt I've been
duplicating many times;
efficient, portable, and hassle-free serialization of objects.

I'm also strongly biased towards functional programming, so immutability plays
a big role in my projects.

<!-- more -->

## TinySerializer

Enter [TinySerializer](https://github.com/udoprog/tiny-serializer-java).

It's a project that started out as two interfaces (`Serializer` and
`SerializerFramework`), but recently grew into it's shoes through the
inspiration of projects like
[AutoValue](https://github.com/google/auto/tree/master/value), and
[AutoMatter](https://github.com/danielnorberg/auto-matter).

I realized that annotation processors have access to a tremendous amount of
information about the types and structure of your classes, and that this
information could be used to generate very efficient and readable serializers.

In the following example, I'll be using [Lombok's @Data annotation](https://projectlombok.org/features/Data.html).

So, given a `Person` like the following:

{% highlight java %}
@AutoSerialize
@Data
class Person {
    private final String name;
    private final Optional<Job> job;
}
{% endhighlight %}

And a `Job`:

{% highlight java %}
@AutoSerialize
@Data
class Job {
    private final String name;
}
{% endhighlight %}

The `@AutoSerialize` processor would then generate the following serializer
for `Person`:

{% highlight java %}
@AutoSerialize
class Person_Serializer implements Serializer<Person> {
    private final Serializer<String> s_String;
    private final Serializer<Optional<Job>> s_OptionalJob;

    public Person_Serializer(final SerializerFramework framework) {
        s_String = framework.string();
        s_OptionalJob = framework.optional(new Job_Serializer(framework));
    }

    public void serialize(SerialWriter buffer, Person person) {
        s_String.serialize(buffer, person.getName());
        s_OptionalJob.serialize(buffer, person.getJob());
    }

    public Person deserialize(SerialWriter buffer) {
        final String v_name = s_String.deserialize(buffer);
        final Optional<Job> v_job = s_OptionalJob.deserialize(buffer);
        return new Person(v_name, v_job);
    }
}
{% endhighlight %}

The use of `SerializerFramework` is an important detail, this gives us _a lot_
of freedom in deciding the exact details of how this serialization is to
operate.

In the serializer, there are no concrete implementations or static
initialization. Everything is Plain Old _Boring_ Java.

You activate the serializer would be to add the following snippet to your maven
dependencies:

{% highlight xml %}
<dependencies>
  <dependency>
    <groupId>eu.toolchain.serializer</groupId>
    <artifactId>tiny-serializer-api</artifactId>
    <version>${tiny.version}</version>
  </dependency>
  <dependency>
    <groupId>eu.toolchain.serializer</groupId>
    <artifactId>tiny-serializer-processor</artifactId>
    <version>${tiny.version}</version>
    <scope>provided</scope>
  </dependency>
</dependencies>
{% endhighlight %}

## Implementation Details

So the above provides you with essentially _free_ object graph serialization
for _your_ objects, but what about primitive types and containers?

To this end, I've built a small companion implementation in
`tiny-serializer-core` which provides you with a relatively sane
`SerializerFramework` implementation.

{% highlight xml %}
<dependency>
  <groupId>eu.toolchain.serializer</groupId>
  <artifactId>tiny-serializer-core</artifactId>
  <version>${tiny.version}</version>
</dependency>
{% endhighlight %}

With it, you can build a new `SerializerFramework` and use it with your class
like this:

{% highlight java %}
public class Application {
    public static void main(String argv[]) {
        final SerializerFramework framework = TinySerializer.builder().build();
        final Serializer<Person> person = new Person_Serializer(framework);

        try (final SerialWriter writer = framework.writeStream(new FileOutputStream("person.bin")) {
            person.serialize(writer, new Person("John-John Tedro", Optional.of(new Job("Programmer"))));
        }
    }
}
{% endhighlight %}

You can see this a larger example in action at the [SerializeToFile
example](https://github.com/udoprog/tiny-serializer-java/blob/master/tiny-serializer-examples/src/main/java/eu/toolchain/examples/SerializeToFile.java).

There are also plenty of configuration options available in `TinySerialize`
that you can play around with.

## Final Words

I hope you find it useful!

If you want to know more, go to [udoprog/tiny-serializer-java](https://github.com/udoprog/tiny-serializer-java).
There is plenty more to read and play with there.

I also intend to eventually implement field-value based serialization that
allows for out-of-order fields.
This is the basis for semantically versioned objects with forward
compatibility, and I consider it an important omission.

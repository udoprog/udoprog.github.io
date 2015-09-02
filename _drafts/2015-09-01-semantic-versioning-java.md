---
layout: post
title: "Semantic Versioning and Java"
description: ""
category: programming
tags: [java]
---

In this post, I'll talk at length about [semantic versioning](http://semver.org/),
and how I believe it should be applied in practice for the long-term maintenance
of Java libraries.

<!-- more -->

Let us introduce the basic premise of semantic versioning (borrowed from their
page), namely version numbers and the connection they have to the continued
development of your software.

1. MAJOR version when you make incompatible API changes,
2. MINOR version when you add functionality in a backwards-compatible manner,
   and
3. PATCH version when you make backwards-compatible bug fixes.

# Hello Java

Let's build an API.

{% highlight java %}
package eu.toolchain.mylib

/**
 * My Library.
 *
 * @since 1.0
 */
public interface MyLibrary {
    /**
     * Do something.
     */
    public void doSomething();
}
{% endhighlight %}

Consider `@since`, it does't contain the patch version. It could, but it would
be useless information.
A patch should _never_ introduce API, that privilege is left to the _minor_
version.

The Java ecosystem would not work without Maven, and the way you _could_ expose
your library is by putting the above in an API artifact named
`eu.toolchain.mylib:mylib-api`.

Eventually, you might also feel compelled to provide an implementation, this
could be distributed as `eu.toolchain.mylib:mylib-core`.

The intent will be to have your users primarily interact with your
library through interfaces, abstract classes, and [value
objects](https://en.wikipedia.org/wiki/Value_object).

# A Minor Change

Let us introduce a minor change to our library.

{% highlight java %}
package eu.toolchain.mylib

public interface MyLibrary {
    /* .. */

    /**
     * Do something else.
     *
     * @since 1.1
     */
    public void doSomethingElse();
}
{% endhighlight %}

In library terms, you are exposing another symbol, or for Java, this is simply
another method, with a given signature added to the already existing
`MyLibrary` interface.
This only constitutes a _minor_ change because consumers if the API happened to
use `1.0` will happily continue to operate in a runtime containing `1.1`.
Anything linked against `1.0` will be oblivious to the fact that there are
added functionality in `1.1`.

Removing a method and not fixing all callers of it would cause
[`NoSuchMethodError`](http://docs.oracle.com/javase/8/docs/api/java/lang/NoSuchMethodError.html)
when a caller with an outdated version attempts to call the missing method.

One of the harder aspects of a minor change is _identifying_ them.
It requires a fair bit of knowledge in how binary compatibility works.

The Eclipse project has compiled
[an excellent page](https://wiki.eclipse.org/Evolving_Java-based_APIs_2)
on this topic which touches a few more cases.
For all the gritty details, you should consult
[Chapter 13](https://docs.oracle.com/javase/specs/jls/se8/html/jls-13.html)
of the
[Java Language Specification](https://docs.oracle.com/javase/specs/jls/se8/html/index.html).

I'll touch on a few things that _are_ compatible, and why.

#### Increasing visibility

Increasing the visibility of a method is a _minor_ change.

Visibility goes with the following modifiers, from least to most visible:

* `private`
* _package protected (no modifier)_
* `protected`
* `public`

From the perspective of the user, a think is not part of your public API if it
is not visible.

#### Adding a method

This works, because method invocations _only_ consult the signature of the
method being called, which is handled indirectly by the virtual machine who
is responsible for looking up the method at runtime.

So this is good _unless_ the client implements the given API.

{% highlight java %}
package eu.toolchain.mylib

/**
 * ... boring documentation ...
 *
 * <em>avoid using directly</em>, for compatibility extend one of the provided
 * base classes instead.
 *
 * @see AbstractMyCallback
 */
public interface MyCallback {
    /**
     * @since 1.0
     */
    public boolean checkSomething();

    /**
     * Oops, sorry client :(
     *
     * @since 1.1
     */
    public boolean checkSomethingElse();
}
{% endhighlight %}

If you are exposing an API that the client should implement, a very popular
compromise is to provide an abstract class that the client _must_ use as
a base to maintain compatibility.

{% highlight java %}
/**
 * A base implementation of {@link MyCallback} that will maintain compatibility
 * for you.
 */
public abstract AbstractMyCallback implements MyCallback {
    /**
     * Should be implemented by client, but if they are using a newer version
     * of the library this will maintain the behavior.
     */
    @Override
    public boolean checkSomethingElse() {
        return false;
    }
}
{% endhighlight %}

You as a library maintainer must maintain this class to make sure that between
each minor release it does not force clients to have to implement methods they
previously did were not required to.

To see this in action, check out
[SimpleTypeVisitor8](https://docs.oracle.com/javase/8/docs/api/javax/lang/model/util/SimpleTypeVisitor8.html)
which is part of the interesting
[java.lang.model](http://docs.oracle.com/javase/8/docs/api/javax/lang/model/package-summary.html)
API.

#### Extending behaviour

This one is tricky, but probably the most important to understand.

If you have a documented behavior in your API, you are not allowed to remove
or modify it.

In practice, it means that once your javadoc asserts something, that assertion
must be versioned as well.

{% highlight java %}
package eu.toolchain.mylib

/**
 * @since 1.0
 */
public interface MyLibrary {
    /**
     * Create a new black hole that will slowly consume the current Galaxy.
     */
    public void createBlackHole();
}
{% endhighlight %}

You may extend it in a manner, which does not violate the existing assertions.

{% highlight java %}
package eu.toolchain.mylib

/**
 * @since 1.0
 */
public interface MyLibrary {
    /**
     * Create a new black hole that will slowly consume the current Galaxy.
     *
     * The initial mass of the black hole will be 10^31 kg.
     */
    public void createBlackHole();
}
{% endhighlight %}

You may not however, change the behavior from `current Galaxy` to `Milky Way`.

{% highlight java %}
package eu.toolchain.mylib

/**
 * @since 1.0
 */
public interface MyLibrary {
    /**
     * Create a new black hole that will slowly consume the Milky Way.
     */
    public void createBlackHole();
}
{% endhighlight %}

Your users will have operated under the assumption that the `current` galaxy
will be consumed.

Imagine their surprise when they run the newly upgraded application in the
`Andromeda Galaxy` and they inadvertently expediate their own extinction
because they didn't expect a breaking change in behavior for a _minor_ version
**:/**.

# A Major Change

Ok, so it's time to rethink your library's existence.
The world changed, you've grown and realized the errors of your way.
It's time to fix all the design errors you made in the previous version.

{% highlight java %}
package eu.toolchain.mylib2

/**
 * My Library, Reloaded.
 * @since 2.0
 */
public interface MyLibrary {
    /**
     * Do something, _correctly_ this time around.
     * @since 2.0
     */
    public void doSomething();
}
{% endhighlight %}

In order to introduce a new major version, it is important to consider the
following:

* Do I need to publish a new package?
* Do I need to publish a new Maven artifact?
* Should I introduce the changes using `@Deprecated`?

This sounds rough, but there are a few points to all this.

#### Publishing a new package

To maintain binary compatibility with the previous Major version.

There are no easy take-backs once an API has been published.
You may communicate to your clients that something is deprecated, and it is
time to upgrade.
You cannot force an atomic upgrade.

If you don't do this, and introduce a Major change that cannot co-exist in
a single classpath. They are in for [a world of
pain](https://groups.google.com/forum/#!topic/protobuf/te3wWId9Qyg).

#### Publishing a new Maven artifact

To allow your users to _co-depend_ on the various major versions of your
library.
Maven will only allow one version of a `<groupId>:<artifactId>` combination to
exist within a given build solution.

If you don't change the artifact, changing the package does not matter.
One package or the other will be missing.

#### Using @Deprecated to your advantage

[`@Deprecated`](http://docs.oracle.com/javase/8/docs/api/java/lang/Deprecated.html)
is a standard annotation discouraging the use of the element that is annotated.

This has wide support among IDEs, and will typically show up as a warning when
used.

You can use this to your advantage when releasing a new Major version.

Assume that you are renaming a the following `#badName()` method.

{% highlight java %}
package eu.toolchain.mylib

/**
 * @since 1.0
 */
public interface MyLibrary {
    /**
     * A poorly named method.
     */
    public void badName();
}
{% endhighlight %}

Into `#goodName()`.

{% highlight java %}
package eu.toolchain.mylib2

/**
 * @since 2.0
 */
public interface MyLibrary {
    /**
     * A well-named method.
     */
    public void goodName();
}
{% endhighlight %}

You can go back and release a new _minor_ version of your `1.x` branch
containing the newly named method with a `@Deprecated` annotation.

{% highlight java %}
package eu.toolchain.mylib

/**
 * @since 1.0
 */
public interface MyLibrary {
    /**
     * A poorly named method.
     *
     * @deprecated Will be removed in 2.0 since the name is obviously inferior.
     *             Use {@link #goodName()} instead.
     */
    @Deprecated
    public void badName();

    /**
     * A well-named method.
     *
     * @since 1.1
     */
    public void goodName();
}
{% endhighlight %}

This is an excellent way of communicating what changes your users can expect,
and can be applied to many situations.

# Case studies

* Jackson performed a move between `1.x` and `2.x` from
  [`org.codehaus.jackson`](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.codehaus.jackson%22)
  to
  [`com.fasterxml.jackson.core`](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.fasterxml.jackson.core%22).
  The transition plan can be followed [on their wiki](http://wiki.fasterxml.com/JacksonUpgradeFrom19To20).
* In doing things the hard way we have
  [Lucene Core](https://lucene.apache.org/core/), and their
  [take on compatibility](http://wiki.apache.org/lucene-java/BackwardsCompatibility).
  [Parts of their
  library](https://github.com/apache/lucene-solr/tree/trunk/lucene/core/src/java/org/apache/lucene/codecs)
  use versioned packages in order to allow different implementations to
  co-exist.
  Most compatibility issues are handled by _rarely_ breaking the public API, and
  doing
  [version](https://github.com/apache/lucene-solr/blob/trunk/lucene/core/src/java/org/apache/lucene/util/Version.java)
  detection
  [at runtime](https://github.com/apache/lucene-solr/blob/trunk/lucene/core/src/java/org/apache/lucene/analysis/Analyzer.java#L251)
  to determine which behavior to implement.
* Guava maintains
  [compatibility](https://github.com/google/guava#important-warnings) for a long
  time, and communicate expectations through their
  [@Beta annotation](https://github.com/google/guava/blob/master/guava/src/com/google/common/annotations/Beta.java).
  Unfortunately there are
  [many things using @Beta](https://github.com/google/guava/search?l=java&q=com.google.common.annotations.Beta&type=Code&utf8=%E2%9C%93)
  at the moment, making this a real consideration when using the library.
* I've recentrly encouraged the Elasticsearch project to
  [consider the versioning implications](https://github.com/elastic/elasticsearch/issues/13273)
  of their java client for the impending `2.x` release.

# Project jigsaw

[Project jigsaw](http://openjdk.java.net/projects/jigsaw/doc/quickstart.html)
is an initiative that could improve things in the near future by implementing a
module system where dependencies and versions are more explicit.

The specification will
[not require implementations to support multiple versions](http://mail.openjdk.java.net/pipermail/jigsaw-dev/2015-June/004336.html)
of the same module, but it should be possible to hook into the module discovery
process in a manner that supports it.

# Conclusion

Dependency hell is far from solved, but good practices can get us a long way.

Good luck, library maintainer.
And may the releases be ever in your favor.

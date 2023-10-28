---
layout: post
title: "Becoming a Debian Maintainer"
description: ""
category: debian
tags: [debian, foss]
---

I've recently started looking into how exactly one would start contributing to
such a huge project like Debian.

It was suprisingly difficult to get my bearings, but I'm starting to get more
comfortable with the inner workings of Debian as a Project.

In this Post I will attempt to explain the things I would have liked to be
explained to me when I started out.

<!-- more -->

# Maintainers and Developers

The first point of contact should be the [Debian New Maintainers'
Guide](https://www.debian.org/doc/manuals/maint-guide/), pay specific attention
to the [Getting started The Right
Way](https://www.debian.org/doc/manuals/maint-guide/start.en.html).

Before continuing you should read this chapter.

Here I noticed that there are three roles that sticks out.

An *upstream maintainer* which is the person *currently* maintaining
a program.
A *Debian Maintainer* is a person that has limited upload rights to
the Debian package archive.
And finally a *Debian Developer*, which is a person which has full upload
rights to the official Debian package archive.

The chapter touches briefly on the difference between an *upstream maintainer*
and a *Debian Maintainer*. But what is the practical difference?

The answer can be found on [mentors.debian.net](http://mentors.debian.net).

<blockquote>
<q>
Only approved members of the Debian project (Debian Developers) are granted the
permission to upload software packages into the Debian distribution. Still
a large number of packages is maintained by non-official developers. How do
they get their work into Debian when they are not allowed to upload their own
packages directly? By means of a process called sponsorship. Sponsorship means
that a Debian Developer uploads the package on behalf of the actual maintainer.
The Debian Developer will also check the package for technical correctness and
help the maintainer to improve the package if necessary. Therefore the sponsor
is sometimes also called a mentor.
</q>
</blockquote>

The mentors project will also be your testing ground, a safe place where you
can receive reviews for your work without causing too much mayhem in the rest
of the project, but open enough and tolerant towards newcomers (like me).

# Systems

The community is built around a couple of systems.
The most important (I've found so far) are the following.

## bugs.debian.org

Bug reports are the documentation process for the entire project.
Everything in Debian is tracked by a bug.

The following are the types of bugs you will initially encounter.

* [ITP - Intent to Package](https://wiki.debian.org/ITP) &mdash;
  Signals to the rest of the project that someone intends to package something.
* [RFS - Request for Sponsorship](http://mentors.debian.net/sponsor/rfs-howto)
  &mdash; If you have a package that you want to be uploaded to the Debian Archive.

## Mailing Lists

These are the lists that you should be apart of.

* [debian-mentors](https://lists.debian.org/debian-devel/)
  * Lurk and participate, this is where you interact with mentors and ask silly
    questions. Beware though, you are expected to do your homework.
* [debian-devel](https://lists.debian.org/debian-devel/)
  * Lurk mostly, this is where technical discussions are being performed for
    the project as a whole, they will probably be over your head for now.
* [debian-user](https://lists.debian.org/debian-user/)
  * Lurk and answer any questions that you feel comfortable in answering.

# First Steps

There are loads of ways to contribute to Debian.

You can write documentation, translate, maintain packages.

For me the first step will be too look for either a package that I would like
to maintain.
This can be either an existing, [orphaned package](
https://www.debian.org/devel/wnpp/orphaned) by changing its bugs' O (Orhpan)
status to ITA (Intent to Adopt).
Just make sure you actually do intend to do it.

Then, regardless of if it's a package you've adopted, or a new one, after
you've done the relevant changes and want to have a sponsor, you'd file a RFS
bug [according to this
guide](https://wiki.debian.org/DebianMentorsFaq#How_do_I_get_a_sponsor_for_my_package.3F).

# Links

For more detailed information, read the following pages.

* [Debian New Maintainers' Guide](
  https://www.debian.org/doc/manuals/maint-guide/)
* [mentors.debian.net](https://mentors.debian.net)
* [Debian Developer (wiki)](https://wiki.debian.org/DebianDeveloper)
* [Debian Maintainer (wiki)](https://wiki.debian.org/DebianMaintainer)
  * [Becoming a Debian Maintainer](
    https://wiki.debian.org/DebianMaintainer#Becoming_a_Debian_Maintainer)

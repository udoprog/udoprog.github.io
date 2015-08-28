---
layout: post
title: "Manually Troubleshooting Portage"
description: ""
category: linux
tags: [gentoo, portage]
---

Portage is a layered system.

The most common interface which everyone using gentoo should be familiar with is ```emerge```.
It is _the definitive command-line interface to the portage system_<sup>[1]</sup>.

Sometimes, however, you are troubleshooting that one failing build, and the minute-long wait for portage to calculate dependencies is mostly a hazzle.

<!-- more -->

[1]: http://www.linuxmanpages.com/man1/emerge.1.php

# ebuild

Portage ebuilds, and the corresponding ```ebuild``` comamnd, is used to build and install packages.

The sequence of commands that emerge runs is the following.

+ ```portage <ebuild> compile```
+ ```portage <ebuild> install```
+ ```portage <ebuild> qmerge```

You as a user can run these manually, just find a package and replace the ```<ebuild>``` with the corresponding ebuild that you wish to install.
This would cause the ebuild to be installed in the same manner as ```emerge``` would have done it, with the following exceptions.

+ Dependencies of the package will not be resolved.
+ The package will not be added to the _world_ set (you can avoid this with ```--oneshot```).

Apart from this, the package would be installed on your system as usual.

You as a developer can now inspect each step for errors, and fix the problems you find.

If you decide that the ebuild has to be modified it's equally straight forward.

{% highlight sh %}
#> mkdir -p $HOME/temp
#> cp -R /usr/portage/<category>/<package> $HOME/temp/<package>
... modify ebuild ...
#> sudo ebuild --skip-manifest <modified-ebuild> ...
{% endhighlight %}

This will cause ```ebuild``` to automatically configure a _temporary_ overlay named ```X-<USER>```.

If you later want to find these packages, use _equery_.

{% highlight sh %}
#> equery has repository x-<user>
...
{% endhighlight %}

# Summary

Interacting with portage can feel daunting at first, but the steps you are taking are no different from the ones emerge itself uses when building your system.

That is the power of a layered system.

P.S. &mdash; You are of course still free to thoroughly shoot yourself in the foot, so be a bit careful!

---
layout: post
title: "Dual Booting: Debian Wheezy &amp; Windows 8.1"
description: "Dual Booting: Debian Wheezy &amp; Windows 8.1"
category: system
tags: [windows, debian, uefi]
---

So I recently got myself in a situation where I had to convert a system that booted Windows 8.1 through EFI, and Debian through MBR legacy into grub.

Of course I had been using my Debian partition for a while and was unkeen to get rid of the rather large amount of changes I've already done.

<!-- more -->

# What got me into this mess

I recently acquired a new computer, and being the cheap bastard I am I only
purchased one new SSD for it.
So to get my priorities straight: at home I mostly game.
So this disk was bound to run Windows.

After a pretty happy installation I sadly discovered that my old hard drive was
starting to fail.
It was also causing boot times into windows to increase so I had to get rid of
it.

After about a month of struggling to get a workable Windows environment I've
had enough and ordered another SSD for work.

# Installing Debian the Naive way

Grab an installer image from https://www.debian.org/distrib/

Hopefully you'll end up with an ISO (I like torrents), now burn it onto a USB
stick.

{% highlight sh %}
#> sudo dd if=image.iso of=/dev/<usb> bs=4M
#> sync
{% endhighlight %}

+ Boot into the newly created USB by messing around with your BIOS.
+ Install it on one of your newly purchased SDDs.
+ Hapilly start using your newly installed system.

By now you should have a working system and are wondering what all the EFI fuzz
is about.

# Growing tired of entering BIOS every day

Eventually you should be growing tired of having to go into BIOS to switch
which system to boot into.
In my particular case my [shitty-yet-awesome
keyboard](https://www.trulyergonomic.com/store/index.php) decides to roll
a dice if it should work pre-boot or not.

So lets figure out how to get a Windows entry into GRUB instead.

# Chainloading Windows 8.1

So, lets chainload our Windows 8.1 partition from within GRUB.

What does it look like?

{% highlight sh %}
#> fdisk -l /dev/sdb

WARNING: GPT (GUID Partition Table) detected on '/dev/sdb'! The util fdisk doesn't support GPT. Use GNU Parted.
{% endhighlight %}

Ah right, GPT.
We're gonna have to use gdisk.

{% highlight sh %}
#> gdisk -l /dev/sdb
...
   1            2048          206847   100.0 MiB   EF00  EFI system partition
   2          206848          468991   128.0 MiB   0C01  Microsoft reserved ...
   3          468992         1083391   300.0 MiB   2700  Basic data partition
   4         1083392      1000214527   476.4 GiB   0700  Basic data partition
{% endhighlight %}

Nice, but this is pretty far fetched from plain old MBR partition tables.

Time to mount the ```EFI system partition``` and find the windows OS loader.

{% highlight sh %}
#> sudo mkdir /boot/efi
#> mount /dev/sdb1 /boot/efi
#> find /boot/efi -name bootmgfw.efi
{% endhighlight %}

We'll also need to find the UUID of the EFI partition so that grub knows where to load the OS Loader from.

{% highlight sh %}
#> ls -l /dev/disk/by-uuid | grep sdb1
lrwxrwxrwx 1 root root 10 Oct  4 17:10 52A9-DDD7 -> ../../sdb1
{% endhighlight %}

Using this information, lets setup a custom boot entry for Windows 8.1.

{% highlight sh %}
#> cat /etc/grub.d/40_custom
... snip

menuentry "Windows 8.1" {
    insmod part_gpt
    insmod chain
    search --fs-uuid --no-floppy --set=root 52A9-DDD7
    chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}
{% endhighlight %}

Let's reboot and try it out!

# GRUB - Error: Invalid Signature

So trying out the meny item was unsuccesful.
Grub gives is an error saying ```Error: Invalid Signature```, what the hell?

After some headscratching we remember the Naive debian installation we recently performed.
The GRUB we are using uses _MBR and legacy_ boot.
None of the _fancy EFI stuff_ is available to us, least of all the ability to verify OS Loader signatures.

Crap. Time to catch up.

Luckily Ubuntu is trying very hard to become EFI compliant and have done a ton of investigation.
All of it documented on [their wiki](https://help.ubuntu.com/community/UEFI).

After reading through the available information, the gist of it is.

* You must boot into an EFI 'environment' in order to modify it's variables in NVRAM (what the hell?).<br />
  <em>How this is done depends on your BIOS.</em>
  (Typically the boot entries that can perform this are prefixed with <b>UEFI:</b>)
* You must either have enabled 'Secure Boot' (correctly signed EFI OS Loader), or disable it in whatever manner is available to your BIOS (if possible).
* Lastly, GRUB can only chainload an EFI OS Loader if itself is started through an EFI OS Loader.<br />
  It can however also load legacy MBR systems in this mode.

So our goal is: <b>Install GRUB into the ```EFI system partition``` so it can boot both our legacy MBR disk, and Windows 8.1 through it's EFI OS Loader</b>

# Installing GRUB into the ```EFI system partition```

We will try to do the following.

+ Boot into a linux system through EFI.
+ chroot into our previously installed Debian system.
+ re-install grub into the ```EFI system partition```.

The best live cd I could find to accomplish this was Ubuntu (>= 12.04.2) 64bit.
As per recommendation on their wiki page, this contains the necessary partitions and EFI loader.
You can find one [here](http://releases.ubuntu.com/), (the torrent is the fastest option).
___take care to download a amd64 version___.

Burn it to a USB in a similar manner as before and boot.

When messing around in BIOS, make sure that when you boot into it that it indicates that it is trying to boot into UEFI.

Check for the ___UEFI:___ prefix or some other indicator.

After boot, check that you have access to the EFI firmware.

{% highlight sh %}
#> stat /sys/firmware/efi
  File: ‘/sys/firmware/efi’
  Size: 0               Blocks: 0          IO Block: 4096   directory
...
{% endhighlight %}

Cool, lets setup our chroot (assuming ___/dev/sda1___ is your debian partition).

{% highlight sh %}
#> sudo mkdir /debian
#> sudo mount /dev/sda1 /debian
#> sudo mount --bind /sys /debian/sys
#> sudo mount --bind /proc /debian/proc
#> sudo mount --bind /dev /debian/dev
#> sudo mount --bind /dev/pts /debian/dev/pts
#> sudo chroot /debian
{% endhighlight %}

Now lets reconfigure grub.

{% highlight sh %}
#> sudo apt-get install --reinstall grub-efi-amd64
{% endhighlight %}

And finally, install grub on the Windows disk with the GPT table.

{% highlight sh %}
#> sudo grub-install /dev/sdb
{% endhighlight %}

Time to reboot!

You should now be able to find another ___UEFI:___ partition in BIOS, this time called ___Debian___.

Make this your default.

# Summary

We now have two disks, ___sda___ and ___sdb___.

+ ___sda___ contains Debian, and is partitioned using ___MBR___.
+ ___sdb___ contains Windows 8.1, and is partitioned using ___GPT___ to boot through EFI.
  The ```EFI system partition``` contains OS Loaders for GRUB and Windows.
  GRUB is capable of reading the partition table of ___sda___ and its configuration file.

Good luck with your new dual boot system!

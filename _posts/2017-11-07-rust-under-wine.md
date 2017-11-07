---
layout: post
title: "Rust applications under Wine"
description: ""
category: rust
---

While writing my [last post] I had the need to compile and run some code under Windows.

Being a Linux fanbox, this situation wasn't optimal. Enter Wine.

[last post]: {{ "/rust/2017-11-05/portability-concerns-with-path.html" | prepend: site.baseurl }}

<!-- more -->

Wine is a _fantastic_ system.
With an initial release 24 years ago, it's grown to encompass incredible things like a full
implementation of DirectX 9, providing very compelling gaming performance for Windows-only games on
Linux.

It also behaves like Windows when you run Rust-based applications on it.

This post is a quick tip for how you can setup a flexible environment for compiling and testing
small Rust applications on Linux. That behave like they would on Windows.

# Installation

Install Wine, with whatever your preferred method is.

Under Fedora, you would use DNF:

```bash
$> sudo dnf install wine
```

Download the installer for Rust from <https://www.rust-lang.org/en-US/other-installers.html>

Like version 1.21 (Stable at the time):

```bash
$> wget https://static.rust-lang.org/dist/rust-1.21.0-i686-pc-windows-gnu.msi
```

Note: Make sure that you download the installer for i686 using a GNU toolchain.

Now, run the installer with wine:

```bash
$> wine msiexec /i rust-1.21.0-i686-pc-windows-gnu.msi
```

After the installer is done, you can create the following helper scripts in
`/usr/local/bin/rust-wine`:

```bash
#!/usr/bin/env bash
set -e
base=$1
shift
[[ -z $base ]] && echo "Usage: $0 <command> [args]" && exit 100
exec wine $HOME/.wine/drive_c/Program\ Files\ \(x86\)/Rust\ stable\ GNU\ 1.21/bin/${base}.exe "$@"
```

You might want to modify the path to the Rust installation to suit your needs.

Let's create a simple Hello World and take it for a spin:

```bash
$> cat > test.rs <<ENDL
fn main() {
  println!("dir: {:?}", ::std::env::current_dir().unwrap());
}
ENDL
$> rust-wine rustc test.rs
$> wine test.exe
Hello World
```

Enjoy!

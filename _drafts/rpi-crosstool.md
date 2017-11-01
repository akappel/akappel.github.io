---
layout: post
title: "raspberry-pi-cross-compiling"
date: 2017-10-26 01:24:24 -0500
---

The following will take you down the rabbit hole of cross-compiling a simple binary (written in [Rust][1]) for your Raspberry Pi 2. Using macOS. And a tool called crosstool-NG.

This is equal parts a tutorial and a singular story, for I cannot guarantee that what I've done will work for you. But I will show you the errors I encountered along the way (and how I solved them) in the hopes that it will help you along your own path.

So yeah, where was I? Oh right; I wanted to cross-compile a "Hello, world!" Rust program on my MacBook that runs on my Raspberry Pi. Why not just install the Rust toolchain directly on my Raspberry Pi and build there? A few reasons:

1. Where's the fun in that?
2. Faster compile times (if I ever write something more intensive than a `println!` statement)
3. ???
4. Profit!

Okay, that was a horrible list. At the end of the day, I went through this process because I can. So without further ado, here's how I attacked it.

To start, [install Rust][2] if you haven't already. In a terminal, run:
```
curl https://sh.rustup.rs -sSf | sh
```

From there, let's create our "hello world" project in whatever Rust workspace you choose (for this example, I'll say it's `/Users/akappel/rust/`):
```
cargo new hello --bin
cd hello
```

You can `cargo run` and see it compile and spit out our favorite programming phrase:
```
   Compiling hello v0.1.0 (file:///Users/akappel/rust/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 1.16 secs
     Running `target/debug/hello`
Hello, world!
```

Siiick. But you already knew how to do all this, didn't you? Okay here's where we start getting to the good stuff. We need to now add a new toolchain "target." Specifically the `armv7-unknown-linux-gnueabihf` target. Adding this target will give us a version of the `rust-std` library that allows us to compile binaries for the Pi. To get the target, run:
```
rustup target add armv7-unknown-linux-gnueabihf
```

You can verify that it's ready for use with a `rustup show` (don't mind the few extra toolchains I have installed):
```
Default host: x86_64-apple-darwin

installed toolchains
--------------------

stable-x86_64-apple-darwin (default)
nightly-x86_64-apple-darwin
1.21.0-x86_64-apple-darwin

installed targets for active toolchain
--------------------------------------

armv7-unknown-linux-gnueabihf
x86_64-apple-darwin

active toolchain
----------------

stable-x86_64-apple-darwin (default)
rustc 1.21.0 (3b72af97e 2017-10-09)
```

Huzzah! Now we're gonna run `cargo build` (because we can't `run` an ARMv7 binary on macOS) with our new target:
```
cargo build --target=armv7-unknown-linux-gnueabihf
```

And we get!...oh:
```
Compiling hello v0.1.0 (file:///Users/akappel/rust/hello)
error: linking with `cc` failed: exit code: 1
...
... (long block of output)

error: aborting due to previous error

error: Could not compile `hello`.
```

No dice. But why? I asked as much on the [#rust][3] irc and got my answer: The missing piece was a cross-compiled C linker. You see, the way I understand it is that the `rust-std` library relies on `glibc` for things like syscalls and other low-level stuff (if this is a gross misstatement, please correct me!). In order to compile a Rust binary, one needs the appropriate C toolchain to be present as well. And this is where [crosstool-NG][4] comes into play.

crosstool-NG is in the toolchain building business. We're going to use this tool to build ourselves a toolchain for linking against ARMv7 compatible `glibc`, which will in turn allow us to successfully build our Rust binary.

Start by following [this page][5] for `brew install`ing the listed prerequisites for macOS.

- Installed the brew dependencies
- cloned the ct-ng repo, configured and ran it in a way where it's located in my /dev/ area
- Configured with `ct-ng armv7-rpi2-linux-gnueabihf; ct-ng menuconfig`, pointing the CT_PREFIX to another cool spot
- Ran build `ct-ng build` and saw it bork because of a weird version error during `gettext` build
- Found I can `autoreconf` in the gettext directory, make it all better and rerun ct-ng build
- Build successful
- Add the resultant bin/ to the path
- Add section to .cargo/config:
```
[target.armv7-unknown-linux-gnueabihf]
linker = "armv7-rpi2-linux-gnueabihf-gcc"
```
- Rerun cargo build for arm
- Success!
- scp that bad boy to the RPi, run it, and bask in your glory

[1]: https://www.rust-lang.org/en-US/
[2]: https://www.rust-lang.org/en-US/install.html
[3]: https://www.rust-lang.org/en-US/community.html
[4]: http://crosstool-ng.github.io/docs/introduction/
[5]: http://crosstool-ng.github.io/docs/os-setup/
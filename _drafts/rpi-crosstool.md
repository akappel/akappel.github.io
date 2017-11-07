---
layout: post
title: "Cross-compiling Rust for the Raspberry Pi on macOS"
date: 2017-10-26 01:24:24 -0500
---

The following will take you down the rabbit hole of cross-compiling a simple binary (written in [Rust][1]) for your Raspberry Pi 2. Using macOS. And a tool called crosstool-NG. It's more than a few steps long, so buckle up and let's get started!

> A note: if you're interested in a (hopefully) simpler solution and are comfortable using Docker, you can try following [these instructions][9] for cross-compiling via Docker container. I haven't tried it myself, so I make no guarantees.

I'll assume you've already [installed Rust][2]. Create your "Hello, world!" project in whatever Rust workspace you choos (for this example, I'll say it's `/Users/USER/rust/`):
```shell
cargo new hello --bin
cd hello
```

You can `cargo run` and see it compile and spit out our favorite programming phrase:
```shell
   Compiling hello v0.1.0 (file:///Users/USER/rust/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 1.16 secs
     Running `target/debug/hello`
Hello, world!
```

Great! Here's where we start getting to the good stuff. You need to now add a new toolchain "target" (via `rustup`). Specifically the `armv7-unknown-linux-gnueabihf` target. Adding this target will give you a version of the `rust-std` library that allows you to compile binaries for the Pi. To get the target, run:
```shell
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

Now run `cargo build` (because you can't `cargo run` ARMv7 binaries on macOS) with your new target:
```shell
cargo build --target=armv7-unknown-linux-gnueabihf
```

And you get...oh:
```shell
Compiling hello v0.1.0 (file:///Users/USER/rust/hello)
error: linking with `cc` failed: exit code: 1
...
... (long block of output)

error: aborting due to previous error

error: Could not compile `hello`.
```

No dice. But why? I asked this question on the [#rust][3] IRC channel and got my answer: the missing piece was a **cross-compiling C toolchain**. Specifically, the C linker. You see, the way I understand it is that the `rust-std` library relies on `glibc` for things like syscalls and other low-level stuff (if this is a gross misstatement, please correct me!). In order to cross-compile a Rust binary, one needs the appropriate C toolchain to be present as well. And this is where [crosstool-NG][4] comes into play.

crosstool-NG is in the toolchain building business. You're going to use it to build yourself a toolchain for linking against ARMv7-compatible `glibc`, which will in turn allow you to successfully build your Rust binary for the Pi.

Start by going to the macOS section of [this page][5] for `brew install`ing the prerequisites. Next, [follow the instructions][6] to clone the repo to a good location and `bootstrap` it:
```
cd /Users/USER
git clone https://github.com/crosstool-ng/crosstool-ng
cd crosstool-ng
./bootstrap
```

Now configure the installation and run it. To set where the tool goes on install, I ran `./configure` with the `--prefix` option set to `$PWD` (which should expand to `/Users/USER/crosstool-ng`):
```shell
./configure --prefix=$PWD
make
make install
export PATH="${PATH}:${PWD}/bin"
```

If all things went as expected, you should be able to run `ct-ng version` and verify the tool's ready to go.

Cool! Before you can run the tool, you'll need to configure it to build your ARMv7 toolchain. Luckily, crosstool-NG comes with some preset configurations, namely `armv7-rpi2-linux-gnueabihf`. Run:
```
ct-ng armv7-rpi2-linux-gnueabihf
```

There should be some output indicating that it's now configured for `armv7-rpi2-linux-gnueabihf`.  You just need to tell `ct-ng` where the toolchain ought to go. We do this with `menuconfig`:
```shell
mkdir /Users/USER/ct-ng-toolchains
cd /Users/USER/ct-ng-toolchains
ct-ng menuconfig
```

It can be overwhelming, as there are a *ton* of options, but stick to the `Paths and misc options --->` menu option. Highlight it and hit Enter. 

Under `*** crosstool-NG behavior ***`, scroll down until you see this long string:
```
(${CT_PREFIX:-${HOME}/x-tools}/${CT_HOST:+HOST-${CT_HOST}/}${CT_TARGET}) Prefix directory
```

Let's update it with our custom directory. Hit Enter, delete the contents, and replace it with `/Users/USER/ct-ng-toolchains`. When you're finished, hit Enter to confirm, scroll over and save, and then exit the configurator.

If you've made it this far, pat yourself on the back; you're about to kick-off your toolchain creation! It only takes one command:
```shell
ct-ng build
```

You can step away from you're computer if you want, because this is gonna take a bit (on my Macbook Pro, it took about ~30 minutes). Since there's a few things that could go wrong here, I can't offer many tips for troubleshooting beforehand. But feel free to reach out to me if you can't figure out an issue on your own.

If it worked successfully, huzzah! You should see a great many binaries now in `/Users/USER/ct-ng-toolchains/armv7-rpi2-linux-gnueabihf/bin`, namely `armv7-rpi2-linux-gnueabihf-gcc`.

For cargo to build using your new cross-compiler, you must:

1. add the `bin` folder listed above to your PATH:
```shell
export PATH="${PATH}:/Users/USER/ct-ng-toolchains/armv7-rpi2-linux-gnueabihf/bin"
```
2. update (or create) your global `/Users/USER/.cargo/config` file with:
```toml
[target.armv7-unknown-linux-gnueabihf]
linker = "armv7-rpi2-linux-gnueabihf-gcc"
```

Return to your Rust project and rerun `cargo build`:
```
cd /Users/USER/rust/hello
cargo build --target=armv7-unknown-linux-gnueabihf
```

The output should be something similar to:
```shell
   Compiling hello v0.1.0 (file:///Users/USER/rust/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.85 secs
```

Boom! What a ride. If you've got [SSH configured properly][7] between your Mac and your Pi, you can [SCP your file][8] over and run the binary remotely:
```shell
scp target/armv7-unknown-linux-gnueabihf/debug/hello pi@192.168.3.155:
ssh pi@192.168.3.155 'chmod +x ~/hello && ~/hello'
Hello, world!
```

Congrats, now go out and make some kick-ass Rust applications for your Pi! Thanks for following along and please let me know if you have any questions, comments, or just found this helpful.

-Adrian


[1]: https://www.rust-lang.org/en-US/
[2]: https://www.rust-lang.org/en-US/install.html
[3]: https://www.rust-lang.org/en-US/community.html
[4]: http://crosstool-ng.github.io/docs/introduction/
[5]: http://crosstool-ng.github.io/docs/os-setup/
[6]: http://crosstool-ng.github.io/docs/install/
[7]: https://www.raspberrypi.org/documentation/remote-access/ssh/
[8]: https://www.raspberrypi.org/documentation/remote-access/ssh/scp.md
[9]: https://github.com/Ogeon/rust-on-raspberry-pi

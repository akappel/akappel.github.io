---
layout: post
title: "raspberry-pi-cross-compiling"
date: 2017-10-26 01:24:24 -0500
---

The following will take you down the rabbit hole of cross-compiling a simple binary (written in Rust) for your Raspberry Pi 2. Using macOS. And something called crosstool-NG.

This is equal parts a tutorial and a singular story, for I cannot guarantee that what I've done will work for you. But I will show you the errors I encountered along the way (and how I solved them) in the hopes that it will help you along your own path.

So yeah, where was I? Oh right; I wanted to cross-compile a "Hello, world" Rust program on my MacBook so that it would run on my Raspberry Pi. Why not just install the Rust toolchain directly on my Raspberry Pi and build there? A few reasons:

1. Where's the fun in that
2. Faster compile times (if I ever write something more intensive than a `println!` statement)
3. ???
4. Profit!

That was a horrible list. At the end of the day, I went through this process because I can. So without further ado, here's how I attacked it.

Step 0. Ensure that your performing dev work in a CaseSensitive location. I'll maybe write a quick tutorial on how I have mine set up in another tutorial but for now, here's something I found online.

- Installed the rust toolchain on my macbook
- Created a new Rust proj with `cargo new hello --bin`
- Ran it with `cargo run` just to see it work locally
- Installed the ARMv7 target
- Attempted to build with the new target, see that it fails
- Ask question on #rust irc, see that I still need a linker
- Find crosstool-NG, think "what in the world have I gotten myself into?"
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
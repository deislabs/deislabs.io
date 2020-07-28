---
title: "A Heaping Helping of Stacks"
description: "Stack and Heap allocation problems with Rust"
date: 2020-07-28 00:00:00 +0000 UTC
authorname: "Taylor Thomas"
author: "@_oftaylor"
authorlink: "https://twitter.com/_oftaylor"
image: "images/twitter-card.png"
tags: ["krustlet", "rust"]
---

We recently [landed support](https://github.com/deislabs/krustlet/pull/321) for Windows in
[Krustlet](https://github.com/deislabs/krustlet). Although the final PR was relatively small, there
was a lot of learning behind it. We ended up learning a whole bunch about stack vs heap allocation
in Rust and figured it would be good to share with the world. But as with most learning
opportunities, this one starts with a story.

## A stack overflow

We started off excited to finally get Krustlet building for Windows. We had built it before with
major modifications back with Krustlet 0.1, but now had all the building blocks in place to build it
normally (without an OpenSSL dependency to boot!). After adding some Windows specific build scripts
and files, a simple `cargo build` worked without problems. Success!!!

Or maybe not...Then we tried to actually run the thing:

```powershell
$ .\target\debug\krustlet-wascc.exe <a bunch of config flags>
<stack trace>
thread 'main' has overflowed its stack
```

Um...that isn't supposed to happen. We haven't even run anything yet. Definitely nothing that should
cause an overflow. Can we at least get the help text?

```powershell
$ .\target\debug\krustlet-wascc.exe -h
<stack trace>
thread 'main' has overflowed its stack
```

Ok, well something definitely seems Wrong™. What about the other binary we compile?

```powershell
$ .\target\debug\krustlet-wasi.exe -h
<stack trace>
thread 'main' has overflowed its stack
```

We were completely perplexed. Why would we not get any output from the program before overflowing?
We tried compiling with the `--release` flag and that got us a little more output before exiting
with the same error. We also started looking for some big structs, but didn't see any in our code.

## Getting somewhere

Someone else at the company suggest we trying bumping the stack size. Turns out it really isn't that
difficult to do (please note that if you have a large project, setting `RUSTFLAGS` will make
everything recompile):

```bash
$ export RUSTFLAGS = "-C link-args=-Wl,-zstack-size=<size in bytes>"
$ cargo build
```

In our case, this didn't really help. We had to get to an eye-watering stack size of 4GB to get
things working, so that wasn't going to work. However, this could be useful to you in your project.

Someone else also suggested printing out the type sizes, which also turns out to be fairly simple.
This does require that you have the `nightly` toolchain installed using `rustup`:

```bash
$ export RUSTFLAGS = "-Zprint-type-sizes"
$ cargo +nightly build
<truncated>
print-type-size type: `std::result::Result<(), std::fmt::Error>`: 1 bytes, alignment: 1 bytes
print-type-size     discriminant: 1 bytes
print-type-size     variant `Ok`: 0 bytes
print-type-size         field `.0`: 0 bytes
print-type-size     variant `Err`: 0 bytes
print-type-size         field `.0`: 0 bytes
print-type-size type: `std::sys::unix::process::process_common::ExitCode`: 1 bytes, alignment: 1 bytes
print-type-size     field `.0`: 1 bytes
print-type-size type: `std::fmt::Error`: 0 bytes, alignment: 1 bytes
print-type-size type: `unwind::libunwind::_Unwind_Context`: 0 bytes, alignment: 1 bytes
    Finished dev [unoptimized + debuginfo] target(s) in 1.62s
```

This output is pretty verbose, but useful. If you are looking for large structs, we'd recommend
filtering out any single byte fields using `grep` or any other tool. After we filtered our input,
most of the structs we'd defined were pretty small until we found one anomaly. It turned out that
our config struct that loaded data from command line flags and environment variables was quite large
due to all of the different things it was checking. So we tried to put it in a
[`Box`](https://doc.rust-lang.org/std/boxed/struct.Box.html) so it would be on the heap instead of
the stack. This partially helped, but one of our binaries was still not working. 

Although this was not helpful for us in the end, it is another tool in your toolbox for identifying
large data structures causing you an overflow. These large structures can then be pared down or you
can put it in a `Box` so that it is put on the heap. An important detail to remember here is that
when you first create a struct, it will be on the stack (see [the Rust
docs](https://doc.rust-lang.org/1.30.0/book/2018-edition/ch04-01-what-is-ownership.html) for a more
detailed overview), as [`Box::new`](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.new)
allocates space on the heap _and then_ puts the struct there. So if you allocate a bunch of large
data structures (not in a `Vec`, [as it points to data on the
heap](https://doc.rust-lang.org/std/vec/struct.Vec.html#guarantees)) before boxing, all of those
will be on the stack.

## The solution
So after that whole rigmarole, we still hadn't gotten anywhere. So after a weekend break, we went
back and started looking through our code one more time. Krustlet uses the [tokio](https://tokio.rs)
runtime to run a whole bunch of control loops. Each of these control loops would spawn other tasks
as needed to perform business logic.

Upon investigation, we noticed that we had used the [`futures`](https://crates.io/crates/futures)
crate [`select!`](https://docs.rs/futures/0.3.5/futures/macro.select.html) macro. This allows you to
wait on multiple futures for the first one to return a response. But according to the docs:

> If a similar async function is called outside of select to produce a Future, the Future must be
> pinned in order to be able to pass it to select. 

We had to set up our futures outside of the select and this caused us to "pin" the futures using
`pin_mut!`. What does that mean? Well, we went back to the docs, and sure enough, we found the
problem:

> Pins a value on the stack

Oh...well that explains a lot. Tokio futures generate a lot of other code and scaffolding (out of
necessity) around your actual work function. So things that each future was doing also would end up
bloating stack use. To be clear, we could have also done a `Box::pin`, but it turns out that `tokio`
also has a [`select!`](https://docs.rs/tokio/0.2.22/tokio/macro.select.html) macro that doesn't
require us to pin (although it does do a little bit of pinning magic under the hood). So once we got
rid of the stack pinning, everything started working with no stack overflows!

## So what did we learn here?

Ok, that was a lot of information, so what did we actually learn here? 

One item not directly addressed above is the whole ["with great power comes great
responsibility"](https://en.wikipedia.org/wiki/With_great_power_comes_great_responsibility) around
zero cost abstractions. Rust focuses on having zero cost abstractions across its whole API surface.
This is quite useful as you can feel free to use any of the abstractions without worry for adding
additional overhead or introducing other behavior you didn't expect. However, it also means you need
to be careful and deliberate in your use of every abstraction, especially as a new Rust programmer. 

In our case, the compiler complained and said "Hey! I need this future to be pinned," so we pinned
it. We didn't take the time to understand what the abstraction was doing for us, which in this case
was pinning to the stack. Rust gives us the ability to easily pin things to the stack or allocate
them on heap (i.e. we don't have to go manually allocate the memory ourselves) instead of magically
managing the choice for us. But in this case, we weren't aware of all the repercussions of our
choices and it led us down a rabbit hole. So the lesson to remember here is to remember that "zero
cost" doesn't mean "zero responsibility."

As for the stack overflow we addressed here, if you are having problems, remember to check these
things:

- Do you need to increase the stack size?
- Do you have any large structs? Try printing the type sizes to check
- Are you pinning or allocating things on the stack you didn't mean to?

Hopefully this is helpful to you and helps you avoid possible pitfalls in your own applications.


---
title: "Still Rusting - One Year Later"
description: "The good, bad, and ugly of using Rust after a year"
date: 2020-12-10 00:00:00 +0000 UTC
authorname: "Taylor Thomas"
author: "@_oftaylor"
authorlink: "https://twitter.com/_oftaylor"
image: "images/twitter-card.png"
tags: ["rust", "krustlet"]
---

It has been about a year since the DeisLabs team starting using Rust in a "serious" project. About this time last year, we started work on what became the [Krustlet](https://github.com/deislabs/krustlet) project. Since then, we have been using Rust extensively across our projects and have learned a ton more about the language's strengths and weaknesses. As "Rust After the Honeymoon" posts currently seem to be all the rage, we thought we could contribute a little to the discussion with our experiences writing applications for the cloud world.

This post is organized using the classic (if not tired) good, bad, and ugly structure. In the bad and ugly sections, everything is stated as points of feedback and is not meant as a complaint. The whole point of this post is to go beyond the more superficial parts of the language and into things that really make a difference in our day-to-day programming work. Spoiler alert: We still _really_ like Rust, so all of this is intended as helpful data for those working on the language, to give people new the the language a good idea of some of the things they might run into, and to help others evaluate Rust for their own use. We have tried to incorporate ideas of possible solutions, no matter how vague, to the problems we bring up.

At the very end, we also have a bonus feature about Go and Rust. Given the team's background in many Go projects, we often hear something like this: "Well, what about Go? Do you regret moving to Rust? What do you miss from Go?" Addressing this in context of our discussion of Rust felt like a smart decision. If you don't care about that topic or it doesn't interest you, feel free to skip it.

Now with that out of the way, let's get going!

## The Good

### Traits

First up, let's talk about traits. We have absolutely loved the trait system in Rust. In particular, we really enjoy the conversion and reference traits (e.g. `TryFrom`/`From`, `AsRef`, `FromStr`, `Deref`, etc.). These are great examples of why traits are better than most other interface-style types – because the type itself doesn't have to implement an interface to be used as another type. `FromStr` allows any type to implement a way to parse a bare string into a type (really useful for APIs). Another simple example can be found in the many types that implement `AsRef<[u8]>` or `Deref<Type = [u8]>`. Instead of having some sort of `Bytes` interface that all the types have to implement to be able to do operations as if it was a slice of bytes, you can just automatically pick up the methods from the underlying type. Yes, I know you could embed types or use inheritance, but the elegance of this is quite nice. This also allows me to write custom types and have easy/cheap conversions or references to them from other external types. It leads to generic parameters that look like this:

```rust
// A function that can write anything that can be accessed as bytes:
// write_all("hello!")
// write_all(String::from("hello!"))
// write_all(b"hello!")
fn write_all<T: AsRef<[u8]>>(data: T)

// Or, I can take anything that can be converted to my custom type
fn do_something<T: Into<MyType>>(thing: T)
```

Basically traits allow you to design flexible APIs for users that allow them to latch on to and/or extend the functionality of your code. This leads to my next point – [Serde](https://serde.rs).

### A Love Letter to Serde

Allow us to indulge in a brief love letter to Serde, the much used serialization/deserialization library leveraged across the Rust ecosystem. To us, it is a first-rate product of Rust’s unique combination of features. It leverages macros, traits, and Rust's emphasis on zero cost abstractions to create a library that is powerful, easy to use, and performant. Developers can easily add serialization or deserialization with a simple `#[derive(Serialize, Deserialize)]` and then customize deserialization behaviors with attributes. Even if you have to implement it manually, there are plenty of docs to read. Once those traits are implemented, any serialization format that has a Serde implementation (like JSON, YAML, etc.) can then serialize or deserialize that data.

### Error handling, `Option`, and `Iter`
Another thing high on our "impressive Rust features" list is an amazing set of mapping, unwrapping, and iteration tools. The built in `Result` and `Option` types combined with their various mapping methods (and `if let` or `let thing = match {...}`) makes it easy to handle errors/missing data in an easy to read way. It also nudges you towards clean and readable error handling patterns (like the try `?` operator), which is helpful for people new to the language. On top of the error handling, we have the `Iterator` trait and its associated methods. There are a whole suite of chainable filters, maps, splitting, and zipping methods (similar to how LINQ and functional programming languages handle collections) along with the all-powerful `collect` method. Below is an example from Krustlet that shows unwrapping an optional value and then mapping and filtering from a collection of data:

```rust
fn mount_setting_for(key: &str, items_to_mount: &Option<Vec<KeyToPath>>) -> ItemMount {
    match items_to_mount {
        None => ItemMount::MountAt(key.to_string()),
        Some(items) => ItemMount::from(
            items
                .iter()
                .find(|kp| kp.key == key)
                .map(|kp| kp.path.to_string()),
        ),
    }
}
```

### Enums

We've found Rust enums really expressive and convenient. Rust enums aren't just single values: they can carry associated data. What's more, each variant can have a different data structure (like discriminated unions from other languages), and you can work with these different cases using pattern matching. They're also full-blown types, so you can implement functions and traits on them.

The value of this is that you can bundle a bunch of possible cases into a single type to pass into (or return from) and function. Working with the cases is safe because you don't need to have optional fields that only may apply to certain cases, and you can only access a case's data when the enum matches that case. The case structure also encourages code that processes enums to adopt a clear, regular layout, making for some quite beautiful code:

```rust
pub enum ClientError {
    /// The item already exists
    AlreadyExists,
    /// The error returned when the request is invalid. Contains the underlying HTTP status code and
    /// any message returned from the API
    InvalidRequest {
        status_code: reqwest::StatusCode,
        message: Option<String>,
    },
    /// A server error was encountered. Contains an optional message from the server
    ServerError(Option<String>),
}

pub fn handle_error(e: ClientError) {
    match e {
        ClientError::AlreadyExists => {
            println!("Item already exists")
        }
        ClientError::InvalidRequest { status_code, message } => {
            println!("Invalid request. HTTP code: {}, message: {}", status_code, message.unwrap_or_default())
        }
        ClientError::ServerError(Some(message)) => {
            println!("Server error: {}", message)
        },
        ClientError::ServerError(None) => {
            println!("Server error")
        },
        
    }
}
```

In this example, we created a simple error type and then unwrapped it according to the data contained inside of each variant. The Rust compiler makes sure we handle all variants of the enum, preventing programmer error (likely from getting distracted by a meme someone posted in chat). 

### Grab Bag
- Macros are awesome and allow you to do some powerful things (and clean up code)
- Cargo still has our hearts. It is hands down one of the top dependency manager and build tools we've used
- To quote a coworker: "NO DAMN NULL POINTERS" (emphasis theirs). You explicitly have to label code as `unsafe` to even get them

## The Bad

### Docs and Clarity

As we have been using various crates across the ecosystem, we've found some interesting patterns in documentation. Docs are sometimes unclear on what is happening in the actual code. They describe the functionality well, but we generally have to go digging through the code to find out whether it is truly a zero cost abstraction or if there are possible side effects to what we are doing. When you first start on projects, generally these kinds of details don't matter. But as you start doing things that require more advanced usage, you end up digging under the hood to see what exactly is going on. For example, if we are using a library that writes data to disk, make sure to clarify which methods flush data or close things down.

Related to this, but slightly different, is trait documentation. As users, if we are trying to find out how we can customize behavior, we always end up jumping through a million functions, looking at all the trait bounds, before we can figure out what we need to implement (It also seems to always be a trait imported from _another_ crate). An example of this from some recent work on Krustlet. We were using the [`tonic`](https://docs.rs/tonic/0.3.1/tonic/) crate and implementing a socket listener for the server. We ended up at one of the functions that allows for a custom handler, but that had 3-4 distinct bounds, 2 of which were traits from external crates. We eventually found an example in the crate repo and it wasn't too difficult, but there was no clear documentation what what needed to be implemented without digging more. This experience is _really really frustrating_ for new Rust developers and we've seen this in multiple crates. The suggestion here would be to put a little more polish into describing what precisely needs to be implemented (even if just linking to an example) on functions with multiple trait bounds.

### Missing Pieces

Something to be aware of coming into the Rust ecosystem is that a lot of crates are still missing features. A recent example of this was finding out that there isn't much support for `multipart` content types in HTTP requests except for `multipart/form-data`. This is not meant to be a complaint against any developer of any crate. We think it is more due to Rust still being newly popular and it is something that people should be aware of.

One explicit thing we'd like to call out is the lack of HTTP support in the standard library. Multiple HTTP implementations + multiple async implementations make it much more difficult than most other languages we’ve used. Perhaps it is time to include the `http` and `hyper`/`actix`/etc. crates (or anything similar) in the standard library. We don't expect that to happen overnight, but we think Rust is past due for standardizing on at least _something_ more than we have now. We understand that the first target of Rust is "system developers," but we believe there are a whole bunch of application developers out there who would love to have some proper HTTP support in Rust.

### Collections of Traits

Operating on a collection of things that all implement the same trait is a little noisy for an application developer. Rust’s philosophy is more that it prefers to act generically on structs. This is great at the systems level where you want to know this stuff, but at an application level it makes it more noisy. We [ran into this](https://deislabs.io/posts/a-fistful-of-states/) in Krustlet when writing our state machine. We wanted to recurse through collection of different structs that all implemented our trait. Because recursion requires a concrete type that matches that trait bound, everything else in the whole call stack now had to match those specific concrete types. You can get around this with Boxing, but that is noisy and annoying. To be clear: this isn't so much a problem to be fixed as syntatical sugar to be added. It would be a nice feature add to not have to explictly handle this boxing all up and down the call stack.

### A Word on the Learning Curve

As we've mentioned in various discussions and posts, the learning curve with Rust was one of our initial small difficulties. As we have been working on more libraries meant to be consumed as crates, we have noted that there is an additional learning curve of designing APIs (particularly when working with generics and trait bounds) for ease of public consumption. There is a certain amount of knowledge you have to pick up by trial and error about how to use them in a way that isn't messy. Additional community created training material and documentation may be useful here to help ease this part of the learning curve.

### Grab Bag

- Macros can lead to weird or hard to decipher errors, generally being labeled by the compiler as occurring in some place nowhere near to the actual code causing the issue.
- Although this is a personal preference on our part, we feel that traits can be overused. So many times we'll read library documentation and be like “where in the world is this coming from?” Then we have to dig through a huge stack of traits and implementations scattered across 20 files to figure out what is happening.
- We wish dependency features in Cargo.toml were a bit more concrete and easier to debug. Right now they are arbitrary strings with no enforcement on how they are used. It sure would be nice if we could see a "dependency graph," as it were, of enabled features and where they were enabled at

## The Ugly

We only really had one problem that qualified as "ugly" in our opinion and that is `async`. Rust has made great strides in improving async support and stabilizing the API. However, there are problems that go beyond the competing runtimes that we mentioned before (which is still a problem that we hope can be solved).

First off are the opaque/complex return types littered around everywhere. Some of this, such as `impl Future<Item = ...>` is built into the standard library. Others, like `impl Stream<Item = ...>`, are more crate specific. 80-90% of the time, this is ok. Other times however, this makes it difficult to store something returned from these functions on a field of a struct and makes it effectively impossible to work around the lack of async lambdas.

Another rough edge centers around `async` with traits. One part of this is the continued need for the `async_trait` crate (which is very useful by the way!) because the core language doesn't support async functions in traits. Based on the issues we have read, there are some technical limitations here, but this needs to be built in to the core language. Related to this is the inability to have `impl Trait` returns in trait functions. In our specific case, we were trying to allow an implementor to return anything that is an `AsyncRead`. That could be a file in some cases or a stream of data from a blob store in another. Instead, we have to return a `Box`ed `dyn`ner, which is both noisy and inconvenient (and now uses dynamic dispatch when it didn't necessarily need to).

Last, and probably the most frustrating, is the sheer amount of boilerplate code we end up writing to manually implement things like `AsyncRead`. We'll admit, most basic things just work and you can always resort to something like `spawn_blocking` (which is not always the best pattern). But when you start doing more advanced things, you end up needing to implement some sort of async trait wrapper to make things work. Often this implementation is nothing more than delegating to some underlying implementation. For example, to get Unix Domain Sockets working in Krustlet, we had to [manually implement an `AsyncRead`](https://github.com/deislabs/krustlet/blob/0b99368e91934d489ade88be99188ec632509475/crates/kubelet/src/grpc_sock/unix/mod.rs#L39) _even though it was already an async-ified type_. This is one place where we hope to take inspiration from Go and make it as easy as `go your_fn()` to make something async. We aren't suggesting that it needs to be the same, just that it needs to be made easier.

To be honest, we are not involved enough in the language design side of things to offer useful technical suggestions. It is surprising that a language that has given us so many wonderful features that make it a joy to code has given us such a rough async API. As developers who have come from multiple different languages with easy to use async implementations, we are quite frustrated by the API as it stands now and know that it could be much better. Improving this API will greatly ease frustration for both new and experienced Rust developers.

## Conclusion

After this year of learning and using Rust in all sorts of projects, we find Rust to be an absolute joy to work in and still highly recommend it as a good candidate for building cloud applications, even with the aforementioned rough edges. However, rough edges are to be expected in any language and we are still hopeful that, as a Rust community, we will come up with solutions to the problems we raised here. If you are reading this while making a decision about whether to use Rust for your next project, hopefully it gives you a good idea about what to expect.

We'd also like to give a huge shout out to the Rust community and the authors of the many crates we have used. It has been a welcoming place and we very much appreciate all the effort various maintainers put into their projects. Hopefully this will be helpful to the community and offer a balanced perspective on the strengths and weaknesses of Rust.

{{< what-about-go >}}

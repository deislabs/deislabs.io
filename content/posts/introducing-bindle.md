---
title: "Aggregate Object Storage Explained"
description: "Introducing Bindle"
date: 2020-12-14 00:00:00 +0000 UTC
authorname: "Matt Butcher"
author: "@technosophos"
authorlink: "https://github.com/technosophos"
image: "images/twitter-card.png"
tags: ["wasm", "wasi", "object storage", "microservices", "rust"]
---

DeisLabs has been experimenting with a variety of WebAssembly tools over the last few months. While it might not appear so at first glance, Bindle is another piece of our WebAssembly research. We were excited to discover applications in addition to WebAssembly.

And what is this Bindle thing? It is an aggregate object storage system. This article explains what we're trying to do, how it works, and why we're building it.

## What Is Bindle?

Take a look around your home. Where do you keep your forks? What about your spoons? And that butter knife? They probably all go in the same place, right? What about your socks? Do you store each sock separately? Do you store each pair of socks separately? Or do you put them all in one drawer?

It can sometimes be convenient to store related items together.

This is the core idea behind aggregate object storage. The system is designed to store (loosely) related things together. More importantly, Bindle makes it easier to store and retrieve related collections _as collections_ instead of dealing with each object as an individual thing.

To understand this a little better, consider the normal way we store objects. With systems like Azure Object Storage, Minio, and S3, each object is stored on its own. For each object I want to store, I need to send it individually. And to access a set of related objects, I have to fetch each independently. And more importantly, if there is a relationship between those objects, I have to figure out on my own how to describe and store a description of that relationship.

Bindle works a little differently. In Bindle, objects are stored and retrieved as collections.

## How Does It Work?

To understand Bindle, we'll need to cover a couple terms:

- A _parcel_ is an individual object. Bindle doesn't care what a parcel is. It could be a GIF, a Word document, an executable binary or something else entirely.
- A _label_ is a small piece of descriptive metadata that describes a parcel.
- An _invoice_ is a document that lists a set of associated parcels. 
- A _bindle_ is an invoice together with all of its parcels. When we talk about the project, we say _Bindle_. When we talk about the object, we say _bindle_.

Both labels and invoices are text documents. We chose to express them using TOML because it's so easy to read.

Here's a simple `invoice.toml` with some comments to explain what it does. It is intended to model a simple website with two files.

```toml=
# This is required for all Bindle documents.
# It's always 1.0.0
bindleVersion = "1.0.0"

# The [bindle] sections describes the aggregate.
# In this case, we're modeling a website with only an
# index.html and a logo.png.
[bindle]
# We always need a name and version. There are other possible
# fields, though, like description and authors.
name = "example.com/boring-website"
version = "1.2.3"

# Next, we declare a couple of parcels. One is for
# the index.html, and the other is for logo.png

[[parcel]]
# The parcel label holds all the information about
# the parcel.
[parcel.label]
# This is a SHA256 of the contents of index.html,
# truncated for readability.
sha256 = "18b9594584801f827a..."
# This is the name of the parcel.
name = "index.html"
# This is the size in bytes of the parcel.
size = 3_866
# This is the media type (a.k.a. content type,
# a.k.a. MIME type) of the file
mediaType = "text/html"

# Here's the second parcel that describes logo.png
[[parcel]]
[parcel.label]
sha256 = "55cbe8a0185dbcd..."
name = "logo.png"
size = 55_441
mediaType = "image/png"
```

The invoice above describes a simple website with only a couple of files. But it should be clear how this could be extended to hold many more files.

From here, we can take a look at how aggregate object storage works in action.

### Storing and Retrieving Bindles

Say you want to store this website bindle on a remote Bindle server. When you push the bindle, the bindle client first sends the invoice. Because SHAs are cryptographically secure, have a strong likelihood of uniqueness, and [here describe the actual content of the parcel](https://en.wikipedia.org/wiki/Content-addressable_storage), we can do something cool. When the server receives the `invoice.toml` from the client, it can check its storage to see whether it already has any of the parcels with the given `sha256` hashes.

If the server does have a parcel with that hash, then the server knows that it already has that exact file. So it can tell the client "don't bother resending that data."

Let's consider what that means in the example above. If I modify my `index.html` file, then build a new bindle for version `1.2.4`, I can send the new `invoice.toml` to the server. At this point, the server will respond with a message that says, "Hey, I already have `logo.png`, so skip that. But please send the new `index.html` because I don't have that SHA-256 in my database."

The same process happens when a client requests a bindle from the server. It fetches the `invoice.toml`, compares the list of parcels to its own parcel cache, and then requests _just the parcels it does not already have_.

In our trivial example above, this may not make much of a difference. But with bindles that have large parcels or dozens and dozens of parcels, this optimization rapidly adds up. It's a faster experience for the user and a cheaper hosting scenario for the provider (less storage space, less bandwidth consumption).

### Grouping and Features

One additional strength of Bindle is that collections can be more sophisticated than just a flat list of parcels. Let's return to our opening example about silverware drawers.

On a Monday morning when I'm having a bowl of oatmeal, I may just grab a spoon from the silverware drawer. On a normal evening, I grab a fork, spoon, and knife. For a fancy formal occasion, I may add the dessert fork and the soup spoon.

The point here is that I might pull out different _configurations_ of silverware for different needs. If my silverware drawer could support only one configuration, I'd have to go with the standard fork, spoon, and knife setup for every situation. I'd have too many utensils for breakfast, and too few for that fancy dinner party. How gauche!

Bindle is designed to handle this situation for data. An invoice can group parcels together. Parcels can be members of different groups. And a Bindle client can then inspect the `invoice.toml` and decide which configuration best suits its present needs.

In a minimalist example, we could model the silverware drawer like this:

```toml=
bindleVersion = "1.0.0"

[bindle]
name = "silverware-drawer"
version = "1.0.0"

[[group]]
name = "breakfast"

[[group]]
name = "regular"

[[group]]
name = "fancy"

[[parcel]]
[parcel.label]
name = "spoon"
[[parcel.conditions]]
memberOf = ["breakfast", "regular", "fancy"]

[[parcel]]
[parcel.label]
name = "fork"
[[parcel.conditions]]
memberOf = ["regular", "fancy"]

[[parcel]]
[parcel.label]
name = "knife"
[[parcel.conditions]]
memberOf = ["regular", "fancy"]

[[parcel]]
[parcel.label]
name = "dessert fork"
[[parcel.conditions]]
memberOf = ["fancy"]

[[parcel]]
[parcel.label]
name = "soup spoon"
[[parcel.conditions]]
memberOf = ["fancy"]
```

> NOTE: The above omits the SHA-256 for each parcel, along with a few other required fields. So this is not a valid invoice.

In the example above, there are  three groups:

- `breakfast`, where I only need a spoon
- `regular`, when a fork, spoon, and knife would be nice
- `fancy`, during which I have lots of flatware

Each parcel declares which groups it is a `memberOf`. The spoon belongs to all three groups, while the dessert fork is only in the `fancy` group.

When a client pulls the invoice above, it is free to simply pull all of the parcels. But it could be more strategic, and say "I only need `breakfast` utensils." By checking that group's membership, it can refrain from pulling those four extra parcels that it does not need. This is particularly helpful if your soup spoon is 40G in size.

## Why Are We Building It?

Bindle was not born out of an abstract sense of architectural purity. That much, I suppose, is obvious from our silverware examples.

We have worked on several package managers and dependency managers. And in a sense, Bindle is a lesson learned from dealing with dependency graphs. We have experimented with modeling Bindle solutions for Node.js and NPM, for Go, and for Helm charts. (We'll release a few examples of this in the next several weeks.)

But what we're really excited about is being able to use Bindle with WebAssembly. Check out Lin Clark's [nanoprocesses model](https://hacks.mozilla.org/2019/11/announcing-the-bytecode-alliance/). Lots of related programs all running together and securely sharing information! This, we think, is the future of cloud computing. But to realize that vision, we're going to need a way to move those aggregate applications around.

We also think that Bindle may make a good fit for [WasmCloud](https://github.com/wasmCloud/wasmCloud) as a way of distributing actors and capability providers. Bindle groups provide a way to enable alternate sets of providers for an actor or actors.

## Jump In!

We've open sourced Bindle because we are deeply invested in moving the WebAssembly ecosystem forward -- and because we think Bindle has some promising applications outside of WebAssembly.

We hope you agree, and we'd love to see you join the conversation. Feel free to [open issues](https://github.com/deislabs/bindle/issues), [create pull requests](https://github.com/deislabs/bindle/compare), or hit us up for information in the [#krustlet channel](https://kubernetes.slack.com/archives/C015X72FV54) on the Kubernetes slack or in the [WebAssembly Discord](https://discordapp.com/invite/nEFErF8).
---
title: "Introducing SpiderLightning - A Cloud System Interface with WebAssembly"
description: "Announcing SpiderLightning and slight, a distributed application runtime for Wasm featuring portable application building blocks."
date: 2022-11-04 00:00:00 +0000 UTC
authorname: "Jiaxiao (Joe) Zhou"
author: "@jiaxiao_zhou"
authorlink: "https://twitter.com/jiaxiao_zhou"
image: "images/logos/twitter-card.png"
tags: ["slight", "spiderlightning", "Wasm", "WASI"]
---

At DeisLabs, we are researching and developing various WebAssembly tools and infrastructure. We have been working on a new project called SpiderLightning, which is a set of distributed application interfaces intended to provide developers in the WebAssembly ecosystem a way to develop applications easily and quickly. These interfaces are written in an Interface Definition Language (IDL) called [WIT](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md), part of the [WebAssembly Component Model](https://github.com/WebAssembly/component-model) work. SpiderLightning is paired with a CLI called slight, which we recently released "0.2.0" version. In this blog post, we will introduce SpiderLightning and slight.

## Problem: Porting distributed applications is hard

Let's set the stage. Imagine you're a developer working at a consumer-facing company. You built a web app that runs on on-premise data centers and used open-source projects to build data stores and pipelines. At the beginning, there are only a few hundreds of customers, so the web app worked well. However, as the company grows, the number of customers increases rapidly. The web app becomes slow and unstable. To get on top of business growth, now you're tasked to migrate the web app to the cloud.

But there is a catch. Migrating to the cloud means that you will need to re-write the code to use cloud resources, such as it's managed databases and pipelines. Even worse, the company you are working on strategically decided to use two cloud vendors to reduce the risk of vendor lock-in. Now you need to re-write the code twice and manage two different deployments. This is a common scenario for many companies. The problem is that the code is not **portable** - the code is strongly coupled with the underlying infrastructure that it runs on.

## Radically Increase Portability with SpiderLightning and Slight

Let's figure out how to make this challenge much easier. Of course, making code more portable has always been a challenge. Think about POSIX. It is a system interface designed to make code portable across different operating systems. The key is to design a system interface that abstracts away specific capabilities into more generically applicable interfaces for those capabilities.

With this in mind, SpiderLightning is a set of distributed application interfaces that are designed to make applications portable across hosting providers (cloud, on-premise, IoT, etc).

Its interface set defines a list of common capabilities that are often needed to build distributed applications. These capabilities include key-value store, blob storage, message queue, transactional database, pub/sub, runtime configuration, distributed lock service etc.

You can think of SpiderLightning as a set of LEGO pieces that can be used to build distributed applications.

![SpiderLightning as lego pieces](/images/spiderlightning/spiderlightning-capabilities.png)

Application developers can then focus on writing business logic and not worry about the underlying infrastructure.

To experiment with SpiderLightning, we built a host CLI called slight. It embeds a WebAssembly runtime called wasmtime and instantiate distributed application capabilities to the underlying infrastructure. For example, it implements key-value capability by using Redis as the underlying data store.

![slight provides services](/images/spiderlightning/slight-services.png)

## How to use slight?

To use slight, you need to install it first. Assuming that you are using a UNIX-based machine, you can install slight by running the following command:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/deislabs/spiderlightning/main/install.sh)"
```

After installing slight, you can create your very first slight application by running the following command:

```bash
slight new -n spidey@v0.2.0 rust && cd spidey
```

This command will create a new Rust project called spidey that will use the SpiderLightning interfaces at version 0.2.0. Other than Rust, we also support starting C projects.

Next, you will need to add `wasm32-wasi` target to your rust toolchain. You can do this by running the following command:

```bash
rustup target add wasm32-wasi
```

Then you can build the project by running the following command:

```bash
cargo build --target wasm32-wasi
```

Finally, you can run the project by running the following command:

```bash
slight -c slightfile.toml run -m target/wasm32-wasi/debug/spidey.wasm
```

If you see "Hello, SpiderLightning!", congratulations! You have successfully run your first slight application.

## How does it work?

You might noticed that slight takes two arguments. The first argument is a configuration file called `slightfile.toml`. The second argument is a WebAssembly module. They represent two phases of software development: operation and development, respectively.

### Operations
Let's take a look at the `slightfile.toml` file:

```toml
specversion = "0.2"

[[capability]]
resource = "kv.filesystem"
name = "my-container"
```

The slight configuration file lists the capabilities that the application needs. In this case, the application needs a key-value store called `my-container`. The key-value store is a filesystem-based key-value store.

Without changing the code, you can change the key-value store to a different infrastructure by changing the configuration file. For example, you can change the key-value store to a Redis-based key-value store by changing the configuration file to the following:

```toml
specversion = "0.2"

[[capability]]
resource = "kv.redis"
name = "my-container"
    [capability.configs]
    REDIS_ADDRESS = "redis://127.0.0.1:6379"
```

Or you can change the key-value store to a Azure BlobStorage-based key-value store by changing the configuration file to the following:

```toml
specversion = "0.2"

[[capability]]
resource = "kv.azblob"
name = "my-container"
    [capability.configs]
    AZURE_STORAGE_ACCOUNT = "${azapp.AZURE_STORAGE_ACCOUNT}"
    AZURE_STORAGE_KEY = "${azapp.AZURE_STORAGE_KEY}"
```

### Development

If you open the `src/main.rs` file, you will see the following code:

```rust
use anyhow::Result;

use kv::*;
wit_bindgen_rust::import!("wit/kv_v0.2.0/kv.wit");
wit_error_rs::impl_error!(kv::Error);

fn main() -> Result<()> {
    let my_kv = Kv::open("placeholder-name")?;
    my_kv.set("hello-spiderlightning", b"Hello, SpiderLightning!")?;
    ...
}
```

The application code imports the key-value store interface from the `wit/kv_v0.2.0/kv.wit` file. It **does not** know, at build time, which key-value store is used. All it knows is that it can use a few operations to interact with the key-value store.

You may wonder what is the `wit/kv_v0.2.0/kv.wit` file. It is a WIT file that defines the interface of the key-value store. It looks like the following:

```go
// A key-value store interface.

resource kv {
	// open a key-value store
	static open: func(name: string) -> expected<kv, error>

	// get the payload for a given key.
	get: func(key: string) -> expected<payload, error> 

	// set the payload for a given key.
	set: func(key: string, value: payload) -> expected<unit, error>

	// delete the payload for a given key.
	delete: func(key:string) -> expected<unit, error>
}
```

Only at runtime, the WebAssembly runtime will provide the implementation of the key-value store interface. At runtime, slight knows which key-value store is used by looking at the configuration file.

## Standardization

Without standardization and community support, SpiderLightning interfaces will not go far. We are working with the WebAssembly community to standardize these distributed application capability interfaces.

Success via standardization will provide a common set of capabilities accessible by **ANY** runtime that exposes those interfaces. It means that you write application code once, and you can run it on any runtime that implement the standard interfaces.

What's more, these interfaces are building on top of the [WebAssembly Component Model](https://github.com/WebAssembly/component-model), which can extend these standard interfaces to even more. For example, you can extend the key-value store interface to add transactional capability.

You may find the proposals [here](https://github.com/WebAssembly/WASI/blob/main/Proposals.md).

Our next step is to work with multiple parties to design a set of interfaces that find the balance between rich feature sets and portability across providers.

## Finishing thoughts

We are excited for the WebAssembly Component Model and WASI to continue maturing. We believe that in the future, you will be able to build portable and secure wasm applications that can run nearly anywhere, including on-premise, cloud, edge, IoT, on satellites in space, anywhere!

Thanks for reading! If you'd like to learn more about SpiderLightning, please check out our [GitHub repo](https://github.com/deislabs/spiderlightning)

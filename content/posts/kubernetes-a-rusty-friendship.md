---
title: "Kubernetes: A Rusty Friendship"
description: "Using Rust with Kubernetes"
date: 2020-04-08 00:00:00 +0000 UTC
authorname: "Taylor Thomas"
author: "@_oftaylor"
authorlink: "https://twitter.com/_oftaylor"
image: "images/twitter-card.png"
---

A few days ago, [we introduced Krustlet](../introducing-krustlet/), a WebAssembly focused
[Kubelet](https://kubernetes.io/docs/concepts/overview/components/#kubelet) implementation in
[Rust](https://www.rust-lang.org). If you are not familiar with Rust, it is a systems programming
language focused on safety, speed, and security. We chose to use Rust for two main reasons: 1) Rust
has some of the best support for WebAssembly compilation (more on this later) and 2) We wanted to
demonstrate Rust and its strengths could be applied to the Kubernetes ecosystem. This post is meant
to show what we learned and why we think Rust is a great (and sometimes better) choice for writing a
Kubernetes focused application.

## An easier developer experience

### Dependency Management
Before even talking about the good coding experience, let's talk about dependency management. As a
bit of background, I am also one of the core maintainers of Helm and have participated in adding new
Kubernetes library dependencies or upgrading the existing ones in almost every release since Helm
2.3. As someone who has developed many large projects against the Kubernetes libraries/API, I can
tell you it is an absolute nightmare to upgrade or add any library dependency. Several of them have
different versioning schemes (such as [`client-go`](https://github.com/kubernetes/client-go)) and
you have to go figure out specific hashes or branches to get everything to compile, particularly if
you have dependencies on multiple libraries. With Rust's powerful dependency and build management
tool `cargo`, things become much more simple:

```toml
kube = "0.31.0" 
k8s-openapi = { version = "0.7.1", default-features = false, features = ["v1_17"] }
```

That is all that is required to pull in most of the same functionality that is spread across
multiple dependencies in Kubernetes Go projects (and sometimes you don't even need `k8s-openapi` if
you aren't constructing the objects). But what is even more powerful and important here is the
ability to conditionally bring in specific parts of a dependency using
[features](https://doc.rust-lang.org/cargo/reference/features.html). In the example above, you can
see us disabling all of the default features and only pulling in the generated objects for version
1.17. We also use this in the [`kubelet`
crate](https://github.com/deislabs/krustlet/blob/cc57726db5971350fe5e3b24476b4b6f712d34d3/crates/kubelet/src/config.rs#L134)
so that users of the crate only pull in a flag parsing library if they explicitly want to do so.

### Ease of coding
Beyond the dependency management, we found that our Rust code was much cleaner than other code we
have written for Kubernetes. Two specific examples of this are the use of macros (one of the
metaprogramming features of Rust) and how Rust deals with error handling. Macros allow you to
automatically generate some code at compile time using the given parameters. Error handling in Rust
uses a type called `Result`, which is an `enum` that indicates whether there was an error or not.
Unhandled `Result`s will result in a compiler warning. These `Result` types come with powerful
matching and chained method expressions to handle things properly. Let's use a concrete comparison
of creating a pod to show the difference:

```rust
let json_pod = serde_json::json!(
    {   
        "apiVersion": "v1",
        "kind": "Pod",
        "metadata": {
            "name": name,
            "labels": user_labels
        },
        "spec": {
            "containers": [
                {
                    "name": name,
                    "image": image
                }
            ]
        }
    }
);

let data = serde_json::to_vec(&json_pod).expect("Should always serialize");
let pod = match client.create(&name, &PostParams::default(), data).await {
    Ok(o) => o,
    Err(e) => {
        error!("Pod create failed for {}: {}", name, e);
        return
    },
}
// Do stuff with pod
```

This example is using the `json!` macro that relies upon the popular Rust deserialization library
[Serde](https://github.com/serde-rs/serde). This macro allows us to inline a JSON object using
previously defined variables. The `let data =` line shows us the use of one of the chainable methods
on `Result` called `expect`. This function will panic with the given message if the Result is not an
`Ok` variant. But most powerfully, we see the use of a `match` statement when we try to patch the
status with the API. This `match` lets us handle the result and unwrap the data inside, performing a
different branch of logic based on the result.

Now let's look at the same functionality in Go:

```go
pod := &v1.Pod{
    Metadata: metav1.Metadata {
        Name: name,
        Labels: userLabels,
    },
    Spec: v1.PodSpec {
        Containers: []v1.Container{
            v1.Container{
                Name: name,
                Image: image,
            },
        },
    },
}

pod, err := client.Create(context.Background(), pod, metav1.CreateOptions{})
if err != nil {
    log.Error("Pod create failed for %s: %s", name, err)
    return
}

// Do stuff with pod
```

Now, these are somewhat similar, but we can see that reading through a pod definition is much more
difficult and it gets more difficult the more there are. Yes, I know there are several methods to
manage this (like a default pod you run a `DeepCopy` on), but the Rust version reads like an actual
pod JSON/YAML you are used to seeing in the Kubernetes world instead of having to manually specify
all of the types. The same thing goes for the error handling. A single `err != nil` doesn't look too
different here, but once we start handling multiple API calls, these `err != nil` blocks keep
multiplying and _they can't be chained_ like `Result`s can.

A concrete example of this can be found in the unit and integration tests for Krustlet. Without an
assert library (like the wonderful [`testify`](https://github.com/stretchr/testify)), Go quickly
becomes stacks of `if err != nil`. With Rust, tests are much cleaner and use a simple
`my_function().expect("oh noes")`.

To put it succinctly, when writing large Kubernetes projects in Go, the code quickly becomes
unwieldy and hard to read. Whereas with Rust, the code stays much cleaner and more readable.

## Extensibility
Another thing we loved about using Rust with Kubernetes was its generics and powerful
[trait](https://doc.rust-lang.org/1.41.0/book/ch10-02-traits.html) system, especially when creating
a Kubelet implementation that others can use and extend. These two features resulted in less code
that was also easily extensible by others. Although what we did is partially replicable using Go
interfaces, it would have been more code across more files (e.g. you can't create default
implementations of functions for an interface) and a whole lot more dynamic dispatch. With generics
in Rust, we could write a Kubernetes client [that accepts any
type](https://docs.rs/kube/0.31.0/kube/api/struct.Api.html#method.create), and interact with it in
the same manner regardless of type. Contrast this to client-go's `PodInterface`, which would limit
us only pods, or we'd need to accept (and implement) the `kubernetes.Interface`, which is quite
massive.

In Krustlet, this allowed us to have a function that could accept any type that can represent a file
path (anything from a bare `str` type all the way up to something more advanced like a `PathBuf`) or
accept any type of `Reader`, while still having concrete types instead of a bare interface. Another
good example of this comes from Matt Butcher's [post on writing controllers in
Rust](http://technosophos.com/2019/08/07/writing-a-kubernetes-controller-in-rust.html). As he
concluded in his post:

> That is all there is to creating a basic controller. Unlike writing these in Go, you won't need
> special code generators or annotations, gobs of boilerplate code, and complex configurations. This
> is a fast and efficient way of creating new Kubernetes controllers.

We observed the same benefits in Krustlet. Having these generics and traits available to us in Rust
allowed us to create software that was much more easily extensible by developers. This empowers
people to extend the Kubernetes control plane in new ways with ease.

## Safety
One of the most touted features of Rust is its safety guarantees. During our time developing
Krustlet, as much as we cursed the borrow checker (sometimes fighting with it for hours when we were
first learning the language), it kept us from exposing possible null pointers, thread safety issues,
and other overlooked bugs. This was particularly helpful as we created the handling for different
Kubernetes Pods.

If you aren't familiar with Kubernetes, a node can run multiple pods, which in turn can run multiple
containers. This means that there was a lot of concurrency and parallel computing going on. Several
times, as we wrote the features for managing pods and "containers" (you don't actually have a
container if you are running WebAssembly module), the borrow checker caught cases of us passing a
data structure between threads that would have resulted in unsafe or concurrent data access
problems. It also prevented us from using objects that could have already been accessed by previous
parts of the code.

For comparison, last week we caught a significant race condition in Helm that has been there for a
year or more, and which passed the `--race` check for Go. That error would never have escaped the
Rust compiler.

The Microsoft Security Research Center wrote a great [blog
post](https://msrc-blog.microsoft.com/2019/07/22/why-rust-for-safe-systems-programming/) on why Rust
eliminates, or greatly reduces the chances of, whole classes of bugs from other languages. Having
this kind of safety built into the language gave us more confidence as the codebase grew. It _did
not_ eliminate all bugs, but we no longer had to worry as much about null pointer exceptions,
concurrency problems, etc. that plagued us in other Kubernetes projects.


## WASM and WASI support
This was a small but important consideration for our use case since we were targeting running
WebAssembly and [WASI](https://wasi.dev) compatible workloads. At this point only a few languages
have WASI support built in, and Rust has some of the best support for WASI workloads right now. It
is as simple as adding the build target like so:

```shell
$ rustup target add wasm32-wasi
```

After that, Rust can build WASI compatible WebAssembly modules with ease. Because of this, many of
the WASM runtimes (such as [`wasmtime`](https://wasmtime.dev), one of the runtimes we used) are
built using Rust, so we could embed them within Krustlet.

If you are considering jumping into WASM/WASI right now, we highly recommend giving Rust a shot.

## The downsides
With all of that said, things aren't always greener on the other side of the fence. There were a few
things we observed that you should be aware of if you decide to do a Kubernetes project in Rust:

- The Rust Kubernetes client library is lacking some of the advanced Go library features (e.g.
  patching, building from a stream of manifests, etc.), and likely will be missing those for the
  next little while.
- Async runtimes are still a bit of a mess in Rust. There are currently 2 different options (`tokio`
  and `async-std`) that have different tradeoffs and problems. Documentation, especially examples,
  is a bit lacking as well. Whereas Go has excellent and easy-to-use concurrency primitives right
  now.
- Rust has a logarithmic learning curve that takes several weeks to get through.

## Conclusion
Although there are a few rough edges, Rust has proved itself as an excellent alternative candidate
for Kubernetes projects. It results in cleaner, more manageable, and easily extensible code that
takes advantage of the added safety benefits provided by the Rust compiler.

If you'd like to give Rust + Kubernetes a whirl, feel free to come check out and/or help with
[Krustlet](https://github.com/deislabs/krustlet); we'd love to see you there!

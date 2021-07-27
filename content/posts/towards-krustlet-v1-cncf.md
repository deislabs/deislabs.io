---
title: "Towards Krustlet 1.0 and CNCF Sandbox"
description: "The good, bad, and ugly of using Rust after a year"
date: 2021-07-25 00:00:00 +0000 UTC
authorname: "Taylor Thomas and Radu Matei"
authorlink: "https://github.com/deislabs"
image: "images/twitter-card.png"
tags: ["rust", "krustlet"]
---

Over a year ago, [we introduced Krustlet][intro], a [kubelet][kubelet]
implementation that enables users to run both WebAssembly and traditional
container workloads in the same Kubernetes cluster. If WebAssembly is new to
you, check out the [section] below.

We started the project with two clear goals in mind:

- make it easy to deploy WebAssembly (and [WASI][wasi]) workloads on Kubernetes
- contribute to the cloud native ecosystem by building core architectural pieces
  of Kubernetes in [Rust][rust], a memory-safe systems programming langauge.

Today we are extremely excited to announce that Krustlet was accepted in the
[Cloud Native Computing Foundation (CNCF)][cncf] as a Sandbox project, alongside
[Krator][krator], a Kubernetes Rust state machine operator framework written in
Rust, as well as the first alpha release of Krustlet, [v1.0.0-alpha.1][alpha1].

This marks the start of API and feature stability for Krustlet, and as we
approach the final 1.0 release, of strong backwards compatibility guarantees.

### The 1.0 roadmap

Krustlet enables developers to build and run WebAssembly and WASI modules on
Kubernetes. Today, you can take advantage of Kubernetes features such as pods,
secrets, init containers, or volume mounts, and in the next couple of weeks, we
will continue to stabilize the current API and feature set. Here are the
specific guarantees for each type of release:

- alpha: No backwards compatibility or API guarantees. However, we do not intend
  to do any major breaking changes, but we reserve the right to should the need
  arise
- beta: Similar to alpha, we will not likely be breaking anything. The one
  exception would be to solve any bugs or missing features found during beta
  releases
- RC: Backwards compatibility and API stability guarantees will go into effect.
  Breaking changes will only be considered for showstopper bugs
- 1.0.0+: No breaking changes will be allowed nor anything that changes
  backwards compatibility. Any breaking changes will be deferred to a future 2.0
  release

> You can get started with Krustlet by following [the official
> documentation][docs].

### What does 1.0 even mean?

"But wait," you ask, "how is this 1.0 when you are still missing things like
full Kubernetes networking?" We chose to move Krustlet to 1.0 because using it
as a Kubelet is now stable enough to actually _build real things_ on top of.
WebAssembly is still rapidly evolving and so there are still rough edges and
other problems to work through. It would be a disservice to the community to not
stabilize what we have so other people can keep learning about and innovating
with WebAssembly. Additionally, there are others in the Rust community who are
building other types of Kubelets that have nothing to do with WebAssembly and we
owe it to them to have a stable version on which to build their own projects.

So, what does this being 1.0 _actually_ mean? First and foremost, it means
stability and backwards compatibility guarantees, as mentioned above. This means
that people can start to build applications built to run on Krustlet (or with
tooling from Krustlet) without fear of a constantly evolving API. This includes
ourselves, where we have plans to create a
[WAGI](https://github.com/deislabs/wagi) Krustlet provider. It also means that
the WebAssembly bits are to the point you can build _something_ with them. Not
everything you can build with containers, but you can build something real. We
will continue to update and add features as WebAssembly and WASI mature.

The tl;dr? The APIs are stable and the project is mature enough to use without
it falling over on itself constantly. WebAssembly? Not fully featured yet, but
ready to be experimented with and used, even if it is a bit bumpy at times.

### WebAssembly, WASI, and containers

[WebAssembly][wasm] (Wasm) is an open standard for a new binary instruction
format, and can used as a compilation target for several popular programming
languages, such as C and C++, Rust, Go (and TinyGo), Swift, or C# (Blazor), as
well as new languages specifically built to target WebAssembly, such as Grain or
AssemblyScript (with multiple ongoing efforts to add support for WebAssembly for
other extremely popular ecosystems, such as Python, with Pyodide, or Kotlin).
Wasm implements a sandboxed execution environment -- specifically, a running
instance has no access to the host runtime's memory or to another instance's
memory, nor does it have access to any host API without the runtime explicitly
providing it.

While originally built for the browser, in essence, WebAssembly provides a fast,
scalable, and portable way of executing the same code regardless of operating
system or CPU architecture. The new [WebAssembly System Interface][wasi] (WASI)
provides a capability-oriented set of APIs designed to standardize the execution
of WebAssembly outside the browser, together with a common layer and set of
primitives that running instances can use to interact with their runtimes, while
maintaining the sandbox promised by WebAssembly, and Krustlet is built on the
reference implementation for WASI, [Wasmtime][wasmtime].

While containers and WebAssembly modules _could_ be used for some of the same
workloads, we believe they are complementary technologies -- WebAssembly's true
OS and architecture portability, compact format, and extremely fast startup,
together with the flexibility of containers as a full operating system sandbox,
used together through Krustlet, allow developers to choose the right tool for
the job.

### Bridging the cloud native and Rust ecosystems

Besides running WebAssembly modules in Kubernetes, the other major goal we set
out for this project was to contribute to the cloud native ecosystem by building
core Kubernetes components in a programming language other than Go.

Krustlet and Krator, together with other libraries and components we are
building with the community (such as a [library for handling OCI artifacts from
Rust][oci-crate]), show how Rust can be a great alternative for building high
performance and secure cloud native components.

{{< krustlet-networking >}}

[intro]: https://deislabs.io/posts/introducing-krustlet/
[kubelet]:
  https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
[releases]: https://github.com/krustlet/krustlet/releases
[wasi]: https://bytecodealliance.org/articles/announcing-the-bytecode-alliance
[rust]: https://www.rust-lang.org/
[cncf]: https://www.cncf.io/
[krator]: https://github.com/krator-rs/krator
[alpha1]:
  https://github.com/krustlet/krustlet/releases/tag/untagged-31951eb32d9122a2afc5
[wasm]: https://webassembly.org/
[wasmtime]: https://github.com/bytecodealliance/wasmtime
[docs]: https://docs.krustlet.dev/krustlet/
[tag]: https://deislabs.io/tags/rust/
[oci-crate]: https://crates.io/crates/oci-distribution
[krs]: https://github.com/kube-rs/kube-rs

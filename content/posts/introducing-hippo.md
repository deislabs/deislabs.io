---
title: "Introduction to Hippo: the WebAssembly PaaS"
description: "Hippo makes it easy to run applications and services at scale"
date: 2021-09-21 00:00:00 +0000 UTC
authorname: "Matthew Fisher"
author: "@bacongobbler"
authorlink: "https://deislabs.io/"
image: "images/twitter-card.png"
tags: ["hippo", "webassembly"]
---

The Deislabs team is really excited about WebAssembly. In fact, we've been
experimenting and sharing our knowledge by developing new technologies in the
WebAssembly space, such as [Krustlet](https://github.com/krustlet/krustlet),
[WAGI](https://github.com/deislabs/wagi),
[bindle](https://github.com/deislabs/bindle),
[yo-wasm](https://github.com/deislabs/yo-wasm), and
[wasi-experimental-http](https://github.com/deislabs/wasi-experimental-http).

While experimenting with these new technologies, it became clear that we need
to make it easy for developers to deploy and test out their new ideas. Today,
we're thrilled to announce a new project. We call it Hippo, an Open Source
self-hosted Platform as a Service (PaaS).

Hippo takes a fresh spin on the PaaS ecosystem, taking advantage of the
technology WebAssembly brings to the space.

Hippo works like this: An application is packaged as a _bindle_. Bindles are
collected together in a [bindle server](https://github.com/deislabs/bindle)
that you can search. Hippo uses `bindle` under the hood for storing and
organizing applications.

Hippo uses a concept called "Channels" to perform automatic deployments of your
applications. More on that later.

Using the `hippo` command line interface, you can upload new releases or
prepare a bindle for local development. In the future, you can use this CLI to
create applications, configure channels, gather logs, attach TLS certificates,
and other commands you'd expect to use with a PaaS.

Hippo provides a web interface for users to register new accounts, access their
applications, and create new environments for testing.

Hippo makes it easy to run applications and services at scale.

## Hippo's Goals

We built Hippo to help with two things.

First, we want to make it simple to develop applications and services. Hippo
provides a suite of tools that make it easy to scaffold and test out new ideas.
We also make it easier for newcomers to start writing apps. **Hippo simplifies
the development experience**.

Second, we want to make it easier for teams to manage their application's
release life cycle. Hippo introduces a concept called "Channels". Channels
automatically deploy the latest release based on the criteria provided. Want to
test your idea in a staging environment? Create a new "staging" channel with a
selection criteria of "\*" and watch Hippo deploy the latest version of your
application right before your eyes. **Hippo simplifies the "go to production"
experience**.

Hippo is a powerful platform. We want to make it easy to develop and manage the
apps and services you deploy.

## What Hippo Brings to PaaS

Hippo uses WebAssembly under the hood as the application runtime.

As a quick refresher, WebAssembly is a low-level assembly-like language with a
compact binary format. It runs at near-native performance, providing languages
with a new compiler target. Hippo utilizes the WebAssembly System Interface
(WASI) to run WebAssembly on the cloud. The [WASI
Overview](https://github.com/bytecodealliance/wasmtime/blob/main/docs/WASI-overview.md)
provides more detailed background information.

WebAssembly provides three main benefits to a platform:

**Hippo is secure**. Applications are sandboxed. Applications have access to
their own memory address space, but they cannot access anything outside its
sandbox environment _unless explicitly granted by the runtime_. This includes
host calls, system files, libraries, or devices.

**Hippo apps are portable**. When compiled to WebAssembly, applications are
unaware of the Operating System's underlying architecture. As a result,
applications deployed to Hippo are portable. If you compile your application
and deploy it to Hippo, it can run on Windows, MacOS, and Linux with no
modification. With Hippo, you can compile and test the exact same binary on
your Windows PC before shipping it to your Linux server. A true "build once,
run anywhere".

**Hippo is fast**. In our experimentation, we found that it takes ~10ms to load
an application and instantiate the WebAssembly runtime _from a cold start_, and
we've been working on new experiments which bring the startup time down to
~3ms.

Hippo takes advantage of WebAssembly to bring a fast, portable, and secure
runtime.

## Closing Thoughts

We set out to build an application platform. We wanted to make it easier for
the developer to run production-grade workloads. We hoped to build something
that seasoned developers will not only use, but contribute to. And we wanted to
make it easier for teams to work together on new projects. We think Hippo is
the right way to do it.

Hippo is ready for you to try out. Check out [the
documentation](https://docs.hippofactory.dev) to get started.

But we're not done yet. We're still working on some things, like expanding the
capabilities of the Hippo CLI. We want to make it easier to run Hippo on
production-grade workload orchestrators like
[nomad](https://www.nomadproject.io/). We are improving Hippo's interactions
with other systems, and growing Hippo's ability to interact with new
WebAssembly features. There is still so much more to come.

But what we're most excited about is the work ongoing in the WebAssembly
community. Hippo's success relies on the success of the community to expand
WebAssembly's horizons and enable users to provide new and exciting use cases.
We're inviting you to jump in and help out.

See you on GitHub!

-- The Deislabs Team

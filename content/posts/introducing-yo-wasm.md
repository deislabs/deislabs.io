---
title: "yo wasm - The Easy Way to WebAssembly"
description: "Introducing a Wasm project generator"
date: 2020-12-08 00:00:00 +0000 UTC
authorname: "Ivan Towlson"
author: "@itowlson"
authorlink: "https://github.com/itowlson"
image: "images/twitter-card.png"
tags: ["wasm", "yo-wasm"]
---

WebAssembly (Wasm) is a portable standard for bytecode, allowing code to be compiled to an efficient representation that's amenable to just-in-time optimisation, and to be run on the operating system and runtime environment of your choice.  An ever-increasing number of languages offer compilation to Wasm, and Wasm runtimes are available in major browsers and as separate programs.  The existence of runtimes outside the browser, such as [`wasmtime`][wasmtime] and [`waSCC`][wascc], opens up the possibility of using WASM as a general-purpose bytecode format, similar to [Java bytecode][javabytecode] or [.NET CIL][cil].  For example, the [Krustlet][krustlet] project provides a way to run WebAssembly modules as Kubernetes pods, performing compute work or serving HTTP requests.

Although languages such as [Rust][rustwasm] and [C/C++][cwasm] can compile to WASM, it's not always obvious how to set up projects in this way - there is extra ceremony compared to most languages' "native" target.  Setting up debugging and deployment also involves extra steps too, and those aren't always obvious.  So there's a barrier to entry in the first place, and unwelcome configuration every time you create a new project.

To make this a bit easier, we're building a Wasm project generator, using the popular [Yeoman][yeoman] code generator, to take care of this setup for you.  We've just released the first preview and we'd love folks to try it out and let us know how it goes.

To install Yeoman and the Wasm generator, you'll need to have Node.js and NPM already installed; then run:

```
npm install -g yo
npm install -g generator-wasm
```

Then generate your new project:

```
mkdir myproject
cd myproject
yo wasm
```

The generator will ask you a few questions, of which two are interesting:

1. Which language do you want the project generated in?  At the moment, we can do Rust, C and AssemblyScript.  We'd be delighted to have more.

2. Do you want to publish the compiled Wasm module to an OCI registry, and if so which one?  This is relevant for workloads that you envisage running in a cloud environment such as Kubernetes with Krustlet.  You don't have to publish to an OCI registry; if you do, we currently only offer Azure Container Registry, but again would love to extend that to other OCI registries.

![Project setup in yo wasm](https://i.imgur.com/QYAQcHH.png)

The result of all this is a "hello, world" application.  The code itself is uninteresting, being just a minimal Rust, C or AssemblyScript program, but the generator also provides a bunch of things to make the development experience easier:

1. Visual Studio Code tasks to build and debug the Wasm build.  This means that - if you're a VS Code user - you can get up and running editing and debugging the project very quickly.  The Debug Wasm debug configuration uses `wasmtime` to run the program, and the LLDB debugger to support breakpoints, etc. in the running Wasm.
    
    ![The Debug WASM configuration in VS Code](https://i.imgur.com/ypz6o0P.png)

2. GitHub actions to build pull requests, and to publish the compiled Wasm module to your chosen OCI registry when you merge to `main` or tag a release with a string of the form `v*` (e.g. `v1.0.0`).  (If you chose not to publish to an OCI registry, this action just creates a build artifact.)
    
    ![Build and publish workflows running out of the box](https://i.imgur.com/ARpY3jl.png)

The preview release has some limitations.  We've mentioned the limited language and registry options.  One important restriction is that all our current templates target WASI (WebAssembly System Interface) and the `wasmtime` runtime.  We'd love to have templates for other environments and runtimes, but we could really do with feedback on that before we invest in it.

We hope you'll give `yo wasm` a try.  Please let us know if you run into any problems by raising an issue at https://github.com/deislabs/generator-wasm/issues, or feel free to send a pull request if there's something you'd like to add or improve.  Thanks!

[krustlet]: https://deislabs.io/posts/introducing-krustlet/
[wasmtime]: https://wasmtime.dev
[wascc]: https://wascc.dev
[javabytecode]: https://en.wikipedia.org/wiki/Java_bytecode
[cil]: https://en.wikipedia.org/wiki/Common_Intermediate_Language
[rustwasm]: https://rustwasm.github.io/docs/book/
[cwasm]: https://developer.mozilla.org/en-US/docs/WebAssembly/C_to_wasm
[yeoman]: https://yeoman.io/

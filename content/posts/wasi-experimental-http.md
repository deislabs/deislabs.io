---
title:
  "An experimental outbound HTTP library for the WebAssembly System Interface"
description:
  "Send HTTP requests from Rust and AssemblyScript Wasm modules running in
  Wasmtime"
date: 2021-03-08 00:00:00 +0000 UTC
authorname: "Radu Matei"
author: "@matei_radu"
authorlink: "https://twitter.com/matei_radu"
image: "images/twitter-card.png"
tags: ["wasm", "wasi"]
---

Over the last year, our team has been experimenting with executing WebAssembly
workloads on the server using the [WebAssembly System Interface][wasi] and
[Wasmtime][wasmtime], and while more and more [proposals][proposals] get
implemented, networking is one scenario that doesn't have a stable API yet,
which limits current applications compiled to WASI, and restricts the types of
workloads that can be executed in WASI runtimes.

Today, [we are releasing a library][gh] which adds experimental and _temporary_
support for outgoing HTTP requests for WASI modules running in Wasmtime. It is
important to note from the beginning that this library does not aim to replace
the WASI sockets API, but rather provide a temporary workaround until the WASI
networking API is stable, and we expect that once [the WASI sockets
proposal][sockets-wip] gets adopted and implemented in language toolchains, the
need for this library would go away.

### Using the new HTTP library

There are two aspects to a WebAssembly library - building guest Wasm modules,
and adding runtime support for instantiating and executing those modules. Let's
first see how to build a guest module in Rust:

```rust
use bytes::Bytes;
use http;
use wasi_experimental_http;

#[no_mangle]
pub extern "C" fn _start() {
    let url = "https://<some-domain>/post".to_string();
    let req = http::request::Builder::new()
        .method(http::Method::POST)
        .uri(&url)
        .header("Content-Type", "text/plain")
    let b = Bytes::from("sending a body in a POST request");
    let req = req.body(Some(b)).unwrap();

    let res = wasi_experimental_http::request(req).expect("cannot make request");
    let str = std::str::from_utf8(&res.body()).unwrap().to_string();
    // do something with the body
}
```

From the start, the one thing to note is that we are reusing the Rust `Request`
and `Response` structures from [the `http` crate][rust-http] (with `Bytes`
request and response bodies), so this way of building HTTP requests should feel
somewhat familiar to Rust developers. Then, a response can be sent and received
using `wasi_experimental_http::request(req)`. This simple program can now be
compiled to the `wasm32-wasi` target. AssemblyScript is another popular language
that natively compiles to WebAssembly, and the library we are releasing also
includes an AssemblyScript NPM package (with examples for how it can be used on
[GitHub][gh]).

In order to run the module in a WebAssembly runtime, we have to satisfy the new
functionality, and we can do this in Wasmtime using the
`wasi_experimental_http_wasmtime` crate:

```rust
use wasi_experimental_http_wasmtime;
use wasmtime::*;
use wasmtime_wasi::Wasi;

let store = Store::default();
let mut linker = Linker::new(&store);
let wasi = Wasi::new(&store, ctx);

// link the WASI core functions
wasi.add_to_linker(&mut linker)?;

// link the experimental HTTP support
wasi_experimental_http_wasmtime::link_http(&mut linker, None)?;
```

The Wasmtime implementation also enables passing a list of allowed hosts - an
optional and configurable list of domains or hosts that guest modules are
allowed to send requests to. If a guest module attempts to send a request to a
domain not explicitly allowed when the runtime was configured, it would receive
an error:

```
'cannot make request: URL not allowed because domain or subdomain not in allowed list
```

Finally, this library works in a similar way to WASI, or to libraries built on
top of WASI: it adds a new Wasm import module, `wasi_experimental_http`, with a
single import function, `req`, which receives the buffer sources for the request
data, performs the request on the host, then writes the response data back to
the guest module's memory, which can then be used by the guest module.

### Support for this library

We are in the process of adding support for this library and outgoing HTTP
requests in [Krustlet][krustlet] and [WAGI][wagi], so keep an eye out in the
respective pull request queues.

We are also happy to help those who want to add support for this library in
their projects.

### Known limitations and contributing

While this library can currently be used to send and receive HTTP data, there
are currently some limitations (for an updated list, please check the issue
queue in the [repository][gh]):

- there is no support for streaming HTTP responses, which this means guest
  modules have to wait until the entire body has been written by the runtime
  before reading it.
- there are no WITX definitions, which means we have to manually keep the host
  call and guest implementations in sync. Adding WITX definitions could also
  simplify adding support for other WASI runtimes (the only WebAssembly runtime
  currently supported by this library is Wasmtime).
- this library does not aim to add support for running HTTP servers in
  WebAssembly.

We welcome all contributions that adhere to the [Microsoft Open Source Code of
Conduct][coc]. Feel free to create pull requests or open issues with any feature
suggestion, and we are happy to chat about the project in the [WebAssembly
Discord server][discord], or in the [ByteCodeAlliance Zulip chat][ba-zulip].

[proposals]: https://github.com/webassembly/proposals
[sockets-article]: https://radu-matei.com/blog/towards-sockets-networking-wasi/
[wasi]: https://wasi.dev/
[sockets-wip]: https://github.com/WebAssembly/WASI/pull/312
[wagi-outbound]: https://github.com/deislabs/wagi/issues/14
[wasmtime]: https://github.com/bytecodealliance/wasmtime
[gh]: https://github.com/deislabs/wasi-experimental-http
[rust-http]: https://crates.io/crates/http
[wasmtime]: https://github.com/bytecodealliance/wasmtime
[krustlet]: https://github.com/deislabs/krustlet
[wagi]: https://github.com/deislabs/wagi
[coc]: https://opensource.microsoft.com/codeofconduct/
[discord]: https://discordapp.com/invite/nEFErF8
[ba-zulip]: https://bytecodealliance.zulipchat.com/

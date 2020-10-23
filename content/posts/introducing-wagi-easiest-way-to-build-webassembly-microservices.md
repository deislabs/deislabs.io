---
title: "Introducing WAGI: The Easiest Way to Build WebAssembly Microservices"
description: "WAGI provides a CGI style interface for writing WebAssembly or WASM microservices"
date: 2020-10-23 00:00:00 +0000 UTC
authorname: "Matt Butcher"
author: "@technosophos"
authorlink: "https://github.com/technosophos"
image: "images/twitter-card.png"
tags: ["wasm", "wasi", "cgi", "microservices", "krustlet", "rust"]
---

A few months ago we released [Krustlet](https://github.com/deislabs/krustlet), a Kubernetes Kubelet that executes WebAssembly payloads instead of Docker containers

Today, we are open sourcing another experimental WebAssembly effort: the [WebAssembly Gateway Interface (WAGI)](https://github.com/deislabs/wagi). Pronounced "waggy" (and inspired by [some](/images/moar_puppy.jpg) of the [puppies](/images/puppy.jpg) on our team), WAGI is the easiest way to build WebAssembly microservices.

WAGI is for writing HTTP response handlers. It uses the [WASI](https://wasi.dev/) POSIX-like system to expose an HTTP request to a WebAssembly module. Rather than requiring developers to learn new frameworks or work directly with network sockets, WAGI uses basic features like environment variables and files.

> What does it mean to be an "experimental" project? DeisLabs does a fair amount of R&D. These projects aren't likely to be the _next Helm_, but we hope that releasing them will be useful to others working in the same space.

## Writing "Hello World" for WAGI

The best way to get started is to look at a simple "Hello World" example. You can write WAGI scripts in any language that can compile to WASM32-WASI. Here we will give an example from Rust.

```rust
fn main() {
    println!("Content-Type: text/plain");
    println!();
    println!("Hello world");
}
```

That's it. You don't need so much as an import. No external dependencies, no elaborate frameworks... just write some data to standard output.

When it comes to WAGI modules, there are only a few things to know:

- Write output to STDOUT
- Read environment variables for request information
- Accept uploads through STDIN

### Writing Output

Sending data to a web client is as easy as writing to `STDOUT`. Like a regular HTTP message, there are two parts to a WAGI response:

- A set of headers, which must contain either a `content-type` or a `location`.
- A body

Between the two is an empty line. Our Rust code above sent this to `STDOUT`:

```
Content-Type: text/plain

Hello world
```

While there are a few other headers you can set, the only one you need is `Content-Type` (which is not case-sensitive).

In most languages, writing to standard output is built in to the core libraries, making it easy to write responses.

### Using Environment Variables

An incoming HTTP request also has headers. A typical HTTP request looks something like this:

```
GET /hello?greet=matt HTTP/1.1
Host: foo.example.com
User-Agent: curl/7.64.1
Accept: */*

```

When WAGI gets a request, it parses the header into environment variables. Here's a quick WAGI module that prints out all of the environment variables. Again, it's in Rust, but we could use other languages to do this.

```rust
fn main() {
    println!("Content-Type: text/plain");
    println!("\n### Env Vars ###");
    std::env::vars().for_each(|v| {
        println!("{} = {}", v.0, v.1);
    });
}
```

The output of the above will show a list of environment variables, which will look something like this:

```
HTTP_HOST = foo.example.com
SERVER_NAME = foo.example.com
SERVER_SOFTWARE = WAGI/1
GATEWAY_INTERFACE = CGI/1.1
HTTP_USER_AGENT = curl/7.64.1
X_FULL_URL = http://foo.example.com/hello?greet=matt
REQUEST_METHOD = GET
SERVER_PROTOCOL = http
PATH_INFO = /env
HTTP_ACCEPT = */*
SERVER_PORT = 80
PATH_TRANSLATED = /env
QUERY_STRING = greet=matt
```

Note that WAGI did quite a bit of processing for you. Frequently, a web app needs to parse the URL or get the HTTP method. WAGI breaks down the information and puts them in separate environment variables so that you don't need to worry about it.

Then, from inside a WebAssembly module, you can just use the built-in language features to access the environment variables. For example, in [AssemblyScript](https://www.assemblyscript.org) (a relative of TypeScript) your code might look something like this:

```typescript 
// Get a handle to the environment variables.
let env = new Environ();

// Get the 'Host' header from the HTTP request.
let host = env.get("HTTP_HOST");
```

A similar feature in Rust would be:

```rust
let host = std::env::var("HTTP_HOST").unwrap();
```

Environment variables are a great way to pass information into a WebAssembly module because they are so easy to access using the libraries that ship with our programming languages.

### Accepting Uploads through `STDIN`

When an HTTP client sends a `POST` or `PUT` request, the client sends a message in the request body. Sometimes this is text. Other times it is a binary file, such as an image.

With WAGI, this information is sent to the WebAssembly module on Standard Input (`STDIN`). `STDIN` is a special file handle. It can be read using the normal file utilities.

Here is a simple Rust WAGI module that echos back the content type and the file contents that the client sent:

```rust
use std::env::var;
use std::io::{copy, stdin, stdout};

fn main() {
    // Get the content type from the request and echo it
    let default_type = "text/plain".to_string();
    let content_type = var("HTTP_CONTENT_TYPE").unwrap_or(default_type);

    // Echo the message back to the client
    println!("content-type: {}", content_type);
    println!();
    copy(&mut stdin(), &mut stdout());
}
```

That's it! In a dozen lines of code using nothing but standard libraries, we built a simple echo WAGI module.

For more examples of WAGI modules, check out the list we maintain in the documentation for [the WAGI server](https://github.com/deislabs/wagi). We also put together a [Rust example module](https://github.com/deislabs/env_wagi) that displays all of the features of WAGI.

## The History of WAGI (Since 1996)

Veteran web developers will take a look at this and recognize a common idiom. WAGI is based on another tremendously popular technology--one that dates back to the birth of the web.

In the mid 1990s, HTML pages were served statically. Authors wrote the HTML document, saved it to disk, and the web server just sent that document directly to the client.

But what if you wanted to execute some logic before sending the page back? That's where Common Gateway Interface (CGI) came in. CGI defined a way to take an inbound request, break it into parts, and feed it to shell scripts. With CGI, one could use Bash, Perl, C-Shell, C, and any other language to write dynamic web pages.

CGI became a _de facto_ standard (and later an [RFC](https://tools.ietf.org/html/rfc3875)), with Apache and other web servers quickly adopting it. Perl and PHP both had early implementations of CGI. And many of the web's first major applications were written as CGI scripts.

As WebAssembly has begun the long maturation process, the WebAssembly Systems Interface (WASI) has a securely sandboxed POSIX environment for WebAssembly modules.

So far, WASI provides access to environment variables, files on the file system (including STDIN and STDOUT), and a few other features. But it has no socket or HTTP support, and it is not yet multi-threaded.

How do you build an HTTP microservice without any networing access? Well, it turns out that CGI provided the answer: We could implement the CGI runtime and pass all the information into the module without needing to directly expose the WebAssembly module to networking. This is secure, convenient, and can be done without WebAssembly multithreading.

The WAGI server, written in Rust, provides a web server that answers requests. On each request, it loads the appropriate WebAssembly module, translates the HTTP request to a CGI request, and then runs the module. All of the threading and state management is handled in the WAGI server. A WAGI WebAssembly module just has to handle a single request.

We are excited about this because we have combined a venerable early web technology and aen emerging technology to provide a simple web service implementation. Will it be the "future of WebAssembly"? Probably not. But it provides for us today a quick and easy way of writing WebAssembly microservices.

## What's the Deal with DeisLabs and WebAssembly?

The DeisLabs team at Microsoft has been increasingly involved in the cloud-native side of WebAssembly. We've been involved with ByteCode Alliance. We've contributed patches and bugfixes to a number of WebAssembly projects. We've written about WebAssembly. And we've released some early experiments.

So what's going on?

We suspect that WebAssembly--specifically with WASI and other projects like waSCC--is poised to become a prominent technology in the cloud native space. Already, over a dozen languages have compilers that can produce WebAssembly binaries.

We see three major boons to WebAssembly.

1. With its low overhead and small footprint, WebAssembly can achieve **higher density in the datacenter**. That means that the cost of cloud hosting goes down.
2. WebAssembly makes a fundamentally positive security assertion: The runtime cannot trust the binary it is executing. That is a perfect model for cloud computing. The default **security posture of WebAssembly is already strong**.
3. Because WebAssembly is **architecture and operating system-neutral**, the same binary (unaltered) can be run on Windows, Mac, and Linux; on Intel, ARM, and other architectures. No more recompiling for specific target platforms.

We think WebAssembly has the potential for a very bright future in the datacenter, in the cloud, and on the edge. In the last few months, we've tried some pretty out-there experiments. Many have failed. But some, like Krustlet and WAGI, are looking promising. We are hoping to build more open source tools for this space, and are eagerly looking forward to cooperating with others in the nascent WebAssembly ecosystem.

We're already toying with some networking, nanoprocesses, and security tools. We hope to share these in the coming months.

We hope you have a good time playing with our WAGI tool. We certainly have enjoyed making it.
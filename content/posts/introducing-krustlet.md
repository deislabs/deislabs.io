---
title: "Introducing Krustlet, the WebAssembly Kubelet"
description: "Announcing Krustlet"
date: 2020-04-06 00:00:00 +0000 UTC
authorname: "Matthew Fisher"
author: "@bacongobbler"
authorlink: "https://twitter.com/bacongobbler"
image: "images/logos/twitter-card.png"
---

Hi everyone, hope you're having a great day today.

At DeisLabs, our goal is to build open-source tools for cloud native developers. We developed a tool
that we think Kubernetes users will enjoy, and the WebAssembly community will be thrilled to hear
about.

Today, we're excited to announce this new project's first release: v0.1.0.

We call it Krustlet, a **K**ubernetes-**rust**-kube**let**.

## Goals of the project

We built Krustlet to help prove out two things:

1. We want to make it easy to deploy WebAssembly workloads on Kubernetes.
2. We want to demonstrate to the community how to build architectural pieces of Kubernetes in
  alternative programming languages. Most programs written for Kubernetes are written in Go. Krustlet
  is written in Rust.

## A comparison of Linux Containers and WebAssembly

Before we can talk about WebAssembly, let's capture a quick summary of Linux containers.

Linux containers (LXC) provide an OS-level virtualized sandbox. In a nutshell, containers allow you
to run multiple isolated Linux systems on a single host. Using certain features from the Linux
kernel, it partitions shared resources (memory, CPU, the filesystem) into isolation levels called
"namespaces". The container runs directly on the physical hardware with no emulation and very little
overhead (asides from a bit of initialization to set up the namespace). The most popular tool using
Linux containers today is [Docker](https://www.docker.com/). Linux containers are different from a
Virtual Machine, where VM management software (VirtualBox, VMware ESXi, etc.) emulates physical
hardware, and the VM runs within that emulated environment.

WebAssembly, on the other hand, is an open standard for a new binary format. By design, it is
memory-safe, portable, and runs at near-native performance. Code from other languages can be
cross-compiled to WebAssembly. Currently, there's first-class support for Rust, C/C++ and
AssemblyScript (a new language built for WebAssembly, compiled against a subset of TypeScript), and
many other compilers are in development.

Originally, WebAssembly was written for the browser. Recent developments have shown that developers
are starting to push WebAssembly beyond the browser. WebAssembly provides a fast, scalable, and
secure way to run the same code across all machines. [WASI](https://wasi.dev/) (the WebAssembly
System Interface) is a new standard that extends WebAssembly's functionality to the OS. By
introducing this new layer of abstraction, code can be compiled to WebAssembly and run anywhere that
supports the WASI standard. A true "compile once, run anywhere".

One might be tempted to compare Linux containers to WebAssembly and pit them against each other.
This would be unfair to both technologies.

Linux containers are designed to provide an OS-level sandbox environment. They rely on the kernel to
provide that sandboxed environment. As a result, code compiled for Intel chipsets cannot run on ARM
hardware.

On the other hand, WebAssembly provides a portable binary format that can run anywhere regardless of
the underlying hardware. However, it is only a binary format. It does not provide the same
flexibility that an OS-level sandbox environment provides.

From our perspective, we see the two technologies as a complementary pair. They each provide their
own set of unique advantages and disadvantages. Together, developers can compile their code and run
it in a container or compile it straight to WebAssembly depending on their application's needs and
requirements.

## Another tool in the toolbox

Here's where Krustlet comes in.

Krustlet is designed to run as a Kubernetes Kubelet. It's similar in design to [Virtual
Kubelet](https://github.com/virtual-kubelet/virtual-kubelet). It listens on the Kubernetes API event
stream for new pods. Based on specific Kubernetes
[tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/), the
Kubernetes API will schedule pods onto Krustlet, which in turn runs them under a WASI-based runtime
(more specifically, either [`wasmtime`](https://wasmtime.dev/) or [`waSCC`](https://wascc.dev/),
depending on which [Runtime
Provider](https://github.com/deislabs/krustlet/blob/master/docs/topics/providers.md) they choose).

Thanks to new, actively developing projects like the WASI standard and waSCC, some codebases can be
compiled to WebAssembly and run on the server.

As mentioned earlier, we see both WebAssembly and Linux containers as a complementary pair. Krustlet
is not intended to replace the Kubelet. When run together, clients can run both traditional Linux
container workloads and WebAssembly workloads in the same cluster. Applications written in
WebAssembly can communicate with applications compiled in Linux containers, and both type of
workloads can interact with the rest of the cluster in a similar manner.

(Tomorrow, we'll post another article about how to think about the project at a high level and
introduce major players in the space. Keep an eye out if you're interested.)

## A Kubernetes component, in Rust

Over the last several months, our team has been writing more and more Kubernetes-specific code in
Rust. Even though Kubernetes itself was written in Go, we are finding that we can typically write
more concise, readable, and stable Kubernetes code in Rust. Matt Butcher wrote an article
demonstrating [how to write a CRD Controller in
Rust](http://technosophos.com/2019/08/07/writing-a-kubernetes-controller-in-rust.html), and Radu has
been writing several articles on [compiling Go libraries as a shared library for re-use across other
languages](https://radu-matei.com/blog/from-go-to-rust-static-linking-ffi/), as well as [writing new
WASI syscalls for wasmtime](https://radu-matei.com/blog/adding-wasi-syscall/).

We took this time to prove that we could achieve something that seemed incredibly difficult a few
months ago: writing projects against the Kubernetes API in languages other than Go.

## Krustlet in action

Assuming we've already got a Kubernetes cluster running and a correctly configured `kubectl`,
installing Krustlet is a piece of cake. Follow [the quickstart
guide](https://github.com/deislabs/krustlet/blob/master/docs/intro/quickstart.md) in Krustlet's
documentation, and you're ready to get started.

Krustlet registers itself as a Kubernetes Node. The first time you run Krustlet, it will
automatically register itself with Kubernetes, ready to schedule WebAssembly workloads:

```console
NAME                                STATUS   ROLES   AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-agentpool-81651327-vmss000000   Ready    agent   3h32m   v1.17.3   10.240.0.4    <none>        Ubuntu 16.04.6 LTS   4.15.0-1071-azure   docker://3.0.10+azure
krustlet                            Ready    agent   11s     v1.17.0   10.240.0.5    <none>        <unknown>            <unknown>           mvp
```

Once it has registered itself with the cluster, feel free to give any of the
[demos](https://github.com/deislabs/krustlet/tree/master/demos/wasi) a try:

```console
$ kubectl apply -f demos/wasi/hello-world-rust/k8s.yaml
```

We can wait for the pod to become ready:

```
$ kubectl wait --for=condition=Ready pod/hello-world-wasi-rust
```

And we can inspect its logs:

```
$ kubectl logs hello-world-wasi-rust
hello from stdout!
hello from stderr!
CONFIG_MAP_VAL=cool stuff
FOO=bar
POD_NAME=hello-world-wasi-rust
Args are: []
```

When we are finished, we can remove it with `kubectl delete`:

```
$ kubectl delete hello-world-wasi-rust
```

## Finishing thoughts

We set out to build a tool that introduces new technology to Kubernetes. We wanted to make it
possible for the Kubernetes developer to run new types of workloads. We hoped to build something
that seasoned developers will not only use, but hopefully take away new learnings from the project.

We have just announced the very first release for Krustlet: v0.1.0. It's ready for you to try out
and experiment. Best of all, it's entirely [open source](https://github.com/deislabs/krustlet). But
we're not done yet. The first iteration was to prove it was possible to run WebAssembly on
Kubernetes. As we look to v0.2.0, we're looking to expand Krustlet's functionality, like supporting
volume mounts, providing more examples, and expanding Krustlet's documentation. We will share more
details in the coming months.

Thanks for reading, and see you on GitHub!

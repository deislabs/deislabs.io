---
title: "Helpful WebAssembly Resources - A List for Kubernetes and WebAssembly"
description: "A useful set of lists for using WebAssembly in Kubernetes instead of containers."
date: 2022-12-05 00:00:00 +0000 UTC
authorname: "Ralph Squillace"
author: "@ralph_squillace"
authorlink: "https://twitter.com/ralph_squillace"
tags: ["kubernetes", "containerd","spiderlightning", "WebAssembly", "spin", "Wasm", "WASI"]
---

Things are moving so fast that often I get asked to host a quick list of the recent things people can read or try for WebAssembly (wasm) outside of the browser, specifically about the WebAssembly System Interface (WASI) and WebAssembly components. Here's some of the best links to various parts of things that did not work so well and those that do. I am leaving out **some** links that we're not pursuing, but at the bottom in the appendix I'll drop a few of those.

In particular, while Custom Resource Descriptions (CRDs) are a great way to integrate things into Kubernetes, our objectives are to simplify using Kubernetes and extend it so that the model you use is less complex. As a result, these links focus on the "heritage" of modeling host processes as "pods" in kubernetes just like containers do.

# WebAssembly 
The core of the work that interests us is the things that will make wasm a "cloud-native" binary that dramatically increases usability and portability but also reduces size and vastly restricts the default security boundaries. Those things are:
- [WebAssembly](https://webassembly.org/specs/). 
- [WebAssembly System Interface (WASI)](https://github.com/WebAssembly/WASI). This is the universal compile target for multiple languages and executable by any WASI-implementing runtime, including capabilities-based security model.
- [WebAssembly component model](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md). The component model enables languages and runtimes to create and consume components freely and still preserve speed, size, and security. 

For a great recent summary of where WASI and components fit, see [Luke Wagner's recent keynote at KubeCon North America in Detroit](https://www.youtube.com/watch?app=desktop&v=phodPLY8zNE). You might also wish to read through some of [Dan Gohman's blog posts on components](https://blog.sunfishcode.online/what-is-a-wasm-component/).

# Projects
Core to using Kubernetes is the Pod concept, always defined as ["one or more containers"](https://kubernetes.io/docs/concepts/workloads/pods/) with shared storage and network resources. We have always wanted Kubernetes users to experience new features and abilities with Kubernetes but barely notice that it's WebAssembly providing those features. As a result, we've focused on enabling wasm runtimes to replace `runc` and other container runtimes while preserving all the other functionality of Kubernetes.

Deislabs started by modeling wasm processes using a [container runtime interface](https://kubernetes.io/docs/concepts/architecture/cri/) or [CRI implementation](https://github.com/deislabs/wok). In theory, WebAssembly is a different form of containment, but in practice `wok` ended rapidly as a project. It was too hard to "fake" all the calls that are essentially hard coded to container-specific concepts. We likely need a CRI v2.0 to move to other "cloud native" process models.
 
Wok was followed by [Krustlet](https://krustlet.dev), a [kubelet implementation](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/). Kubelet's represent a node or machine, so though this project was successful, it marked out a specific node as a "special" wasm-running computer. The metaphorical "cattle" started becoming "pets" again, which was the wrong direction. We wanted to use wasm to make even the "pet" nodes less noticeable, not more. (Krustlet does, however, have a great OCI distribution crate in Rust, which will likely become more important as time goes on.)

We've now settled on a [ContainerD](https://containerd.io/) shim implementation. Specific container runtime controls for kubelet are implemented through the containerd  API. A ContainerD shim is an internal implementation between the ContainerD API and the "container" runtime that can make modifications and perform other tasks before, during, and after a pod has been scheduled. For more on how shims work, see [Shim Shiminey Shim Shiminey](https://container42.com/2022/01/10/shim-shiminey-shim-shiminey/). 

Deislabs originally wrote a shim that uses the [Bytecode Alliance Foundation](https://bytecodealliance.org)'s [wasmtime](https://wasmtime.dev/) runtime that has just moved into the CNCF's ContainerD project, called [runwasi](https://github.com/containerd/runwasi). `Runwasi` does the main bit of work for any shim that wants to layer in a application host that itself consumes `wasmtime` as its runtime. Two of those kinds of shims -- one for [Fermyon](https://fermyon.com)'s `spin` and one for Deislabs's `slight` runtimes -- are the basis for the [Azure Kubernetes Service's WASI NodePool Preview](https://learn.microsoft.com/en-us/azure/aks/use-wasi-node-pools) and the same preview is running on AKS Lite's Linux Preview. You can find those [shims here](https://github.com/deislabs/containerd-wasm-shims), including [quickstart instructions to run them on K3s](https://github.com/deislabs/containerd-wasm-shims/blob/main/deployments/k3d/README.md#how-to-run-the-example). 

This has already shown to have a ton of promise; cold-starting speed is hard to imagine, resource utilization is way up, and the network and disk footprint is way down. Recently, Docker release a technical preview of Docker Desktop that used a version of the `runwasi` shim modified by SecondState to use the wasmedge runtime instead of wasmtime. With the move to the containerd project, we hope to collaborate on enabling a selection of runtimes and add OCI features like OCI Artifacts as well. 

We hope these links help you think more concretely about the features that WebAssembly brings to Kubernetes. If you find them interesting, give [Azure Kubernetes Service's WASI NodePool Preview](https://learn.microsoft.com/en-us/azure/aks/use-wasi-node-pools) a try. You can even create a spin application that runs standalone, in Fermyon's new cloud, and in Kubernetes. 

More coming here soon, including when you want to use a wasm cluster -- and when you don't.
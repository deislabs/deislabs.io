---
title: "Akri a Year Later"
description: "Akri turns One and Enters the CNCF Sandbox"
date: 2021-10-20 00:00:00 +0000 UTC
authorname: "Kate Goldenring"
author: "@kate-goldenring"
tags: ["akri", "cncf"]
---

A year ago, [we announced Akri](https://cloudblogs.microsoft.com/opensource/2020/10/20/announcing-akri-open-source-project-building-connected-edge-kubernetes/), a Kubernetes operator that discovers and exposes IoT devices to clusters. With hard work and a sprinkle of luck over this past year, we were officially accepted into the [Cloud Native Computing Foundation (CNCF) ](https://www.cncf.io/)as a [Sandbox project](https://www.cncf.io/sandbox-projects/)! This milestone marks an opportunity to look back at the motivation behind Akri and forward at what the future holds for Akri.

## Why Akri?
Akri came from a question on how to incorporate constrained devices into Kubernetes clusters, when they may not have enough compute to be Nodes themselves. Akri wanted to solve problems unique to the Edge: discovering devices, the intermittent connectivity of network-based IoT devices, and the sharing of devices by multiple Nodes. These problems all boiled down to one question: Is a IoT device available for my application to use? We decided to make the answer as clear as possible by using the device plugin framework to represent IoT devices as requestable resources. The result is an experience that abstracts away device discovery and intermittent connectivity. So long as the device resource exists, the device can be used. With Akri, applications can declaratively require IP cameras, OPC UA servers, USB thermometers, and more just as they would request for CPU or memory. Akri aims to become the standard way to use IoT devices on Kubernetes clusters on the Edge, which is why Akri not only means “Edge” in Greek but also stands for A Kubernetes Resource Interface. Standards come from community led and owned projects, which is why having a neutral home in the CNCF gets Akri closer to shaping Kubernetes and IoT experiences.

## What’s happening in the Sandbox?
We are very excited to announce that Akri was accepted into the CNCF as a [Sandbox project](https://www.cncf.io/sandbox-projects/)! This shows that Akri solves a unique problem and does it all in a cloud native way. With the acceptance comes support and promotion by the CNCF. Akri will soon be moved from Deis Labs to its new home under the [project-akri](https://github.com/project-akri) organization on GitHub. Thank you to everyone who helped get Akri to this milestone! 

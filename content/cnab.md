---
title: "CNAB"
description: "Cloud Native Application Bundles"
---

[Cloud Native Application Bundles](https://cnab.io) is an open-source package
format specification for managing distributed applications with a single
installable file. With bundles you can reliably provision application resources
in different environments, and manage their application lifecycle, without
having to use multiple toolsets.

There are three distinct pain points that we want to address with CNAB:

1. We need to be able to describe our application as a single artifact, even
   when it is composed of a variety of cloud technologies;
2. We must be able to provision our applications without having to master dozens
   of tools; and
3. We need to manage application lifecycle, particularly installation, upgrade,
   and deletion.

> Bundles are cloud installers for your app and its infrastucture

CNAB relies on a handful of technologies you are already familiar with – JSON
and Docker containers – and describes a format for packaging, installing, and
managing distributed applications. By design, it is cloud agnostic. It works
with everything from Azure to on-premise OpenStack, from Kubernetes to Swarm,
and from Ansible to Terraform. It can execute on a workstation, a public cloud,
an air-gapped network, or a constrained IoT environment. And it is flexible
enough to accommodate an array of platform needs, from customer-facing
marketplaces to internal build pipelines.

Broadly, CNAB brings several features that aren’t currently in the ecosystem:

* Manage discrete resources as a single logical unit that comprises an app.
* Use and define operational verbs for lifecycle management of an app such as
  install, upgrade, uninstall or define your own custom verbs.
* Sign and digitally verify a bundle, even when the underlying technology
  doesn’t natively support it.
* Attest and digitally verify that the bundle has achieved that state to control
  how the bundle can be used.
* Export a bundle and all referenced components to reliably reproduce in another
  environment, including offline environments (IoT edge, air-gapped
  environments).
* Store bundles in repositories for remote installation.

CNAB can build on top of existing technologies and doesn't necessarily have to
be a replacement for what you are using today. If you are already using bash
scripts, Terraform, Docker Compose, and your cloud provider's tooling or a mix
of all of these to manage your application, then your bundles can continue to
use those. Depending on your needs, you can select one of the [CNAB compliant
tools][tools] to create your bundle and try out Cloud Native Application Bundles today.

This is a modified copy of the original announcement [Introducing CNAB: a cloud-agnostic format for packaging and running distributed applications][announcement] by [Matt Butcher][technosophos].

[tools]: https://cnab.io/community-projects/#tools
[announcement]: https://cloudblogs.microsoft.com/opensource/2018/12/04/announcing-cnab-cloud-agnostic-format-packaging-running-distributed-applications/
[technosophos]: https://twitter.com/technosophos
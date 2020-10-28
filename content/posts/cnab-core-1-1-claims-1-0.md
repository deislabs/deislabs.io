---
title: "Announcing CNAB Core 1.1 and CNAB Claims 1.0"
description: "Cloud Native Application Bundles publishes two new specifications"
date: 2020-10-30 00:00:00 +0000 UTC
authorname: "Matt Butcher"
author: "@technosophos"
authorlink: "https://github.com/technosophos"
image: "images/twitter-card.png"
tags: ["cnab", "claims", "specification", "porter"]
---

Cloud Native Application Bundles (CNAB) is one of the projects that DeisLabs works on. Together with Docker, DataDog, and others, we have been defining a specification for cloud-based package management. This is different than something like Helm, which is Kubernetes-specific. CNAB allows you to work with any public cloud or cloud service provider, as well as on-premises/edge systems to provision and configure resources.

The [CNAB project](https://cnab.io) defines the standards that CNAB implementations use. Additionally, it provides reference implementations for some CNAB specifications. Tools like [Porter](https://porter.sh) implement those specifications.

This October, the CNAB project has reached a pair of milestones.

The CNAB working groups have been busy working on their specifications, and we are proud to announce [two completed specs](https://github.com/cnabio/cnab-spec). The CNAB Claims specification has reached its 1.0.0 milestone. And the CNAB Core has gone from version 1.0.1 to version 1.1.0.

In this post, we will take a quick look at what these new releases mean.

## Overview of the CNAB Specification Landscape

CNAB is composed of the following specifications:

- **CNAB Core:** The definition of the CNAB bundle and how such a bundle is to be installed, updated, or deleted.
- **CNAB Security:** The security guidance for signing, transmitting, and verifying bundles.
- **CNAB Registry:** The specification for how bundles are stored in and retrieved from an OCI registry.
- **CNAB Claims:** The definition of a data format for sharing information about specific CNAB installations.

The CNAB Core specification is the only one that is required for a system to be CNAB-conformant. The others cover interoperability between CNAB tools.

## What is the CNAB Claims 1.0 Specification?

CNAB packages (bundles) may install components in a wide variety of destinations -- cloud services, Kubernetes clusters, PaaS platforms, etc. CNAB-aware tools need to be able to determine which things it manages and in which systems. Moreover, multiple CNAB-aware tools may need to share that information.

The [CNAB Claims specification](https://github.com/cnabio/cnab-spec/tree/cnab-claims-1.0.0) describes a data format for sharing installation information, including what CNAB operations were run and what configuration was sent to the invocation image.

The now-deprecated Duffle client introduced Claims prior to its 1.0 release. But the new version of Claims is designed to be used across a wide variety of CNAB tools. The reference implementation of claims is available in [the CNAB-Go Library](https://github.com/cnabio/cnab-go/). Porter already includes Claims 1.0 support.

## What's New in CNAB Core 1.1

[CNAB Core](https://github.com/cnabio/cnab-spec/tree/cnab-core-1.1.0) is a stable specification. Changes are made cautiously. When an ambiguity was discovered in the core specification, it was resolved in a way that we deemed significant enough to warrant a minor release of the specification.

In this case, we discovered that the definition of a `contentDigest` in the `bundle.json` was not sufficient to explain how it should be generated. The language was [clarified](https://github.com/cnabio/cnab-spec/commit/59f80b07cb4efe0f82eedf5ead53a16dc2e8921c). Specifically, the new version makes it clear that `docker` and `oci` image types must use the registry-computed digest for this value (as opposed to a client-side digest).

## Looking Forward

The CNAB team has only one remaining specification to complete: The [CNAB Registry Specification](https://github.com/cnabio/cnab-spec/blob/cnab-core-1.1.0/200-CNAB-registries.md). It is nearing draft completion, and we are excited at the prospect of a complete set of finished specifications.
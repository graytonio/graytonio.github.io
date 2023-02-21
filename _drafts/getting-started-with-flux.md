---
title: Getting Started With FluxCD
date: 2023-02-20
categories: [Homelab, Kubernetes]
tags: [homelab, kubernetes, automation]
---

# Getting Started with FluxCD

[FluxCD](https://fluxcd.io/) is a GitOps deployment system for kubernetes that allows the configuration of a cluster to be tracked in a git repository and applied through managing the main branch.

In this post I'll go over how I think is a good way to setup flux which will allow for a staging to production flow, easily enabling and disabling applications, and easily recovering from hardware failures. All the code for in this post will also be available in my [github]()

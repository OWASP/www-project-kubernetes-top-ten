---

layout: col-sidebar
title: "K07: Misconfigured And Vulnerable Cluster Components"
---

## Overview

Misconfigurations and vulnerabilities in cluster components can leave them exposed to attackers, allowing for easier compromise of applications and the cluster itself. As Kubernetes ships without hardened defaults, it's important to change default settings to improve cluster security.

## Description

There are a number of components which make up a Kubernetes cluster. In addition to the main Kubernetes components (kube-apiserver, controller manager, scheduler, kubelet, and kube-proxy) the configuration of cluster nodes and components like the container runtime can affect cluster security.

Kubernetes components often ship with un-hardened default settings, which could allow attackers easier access to gain initial access or escalate their rights after compromise.

Cluster nodes should also be hardened to improve their security, as attackers could target them as a means to get an initial foothold in a cluster.

Patching is also an important consideration as security vulnerabilities can occur at any layer of the container stack.

## How to prevent

Hardening Kubernetes itself can be done either by the distribution provider or the cluster operator. The CIS benchmark for Kubernetes provides a good starting point for this effort, although it's important to note that not every recommendation will be appropriate for every cluster. Tooling can also be used to automate this process, with projects like Kubescape

Node OS hardening is another important consideration, one option for this is to use a Linux distribution that's focused on running container workloads like [Flatcar Linux](https://www.flatcar.org/) or [Talos Linux](https://www.talos.dev/). Managed Kubernetes providers will also often provide a hardened node OS specific to their cloud environment (e.g. AWS Bottlerocket or Google Container Optimized Linux).

When patching Kubernetes it's important to know the product's support lifecycle. Upstream Kubernetes provides approximately 14 months of support for releases, but some distributions will extend that by providing Long Term Support (LTS) versions of Kubernetes.

## References

- [CIS Benchmark for Kubernetes](https://www.cisecurity.org/benchmark/kubernetes)   
- [Kubernetes support period](https://kubernetes.io/releases/patch-releases/#support-period)  
- [Kubescape](https://kubescape.io/)
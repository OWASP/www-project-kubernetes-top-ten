---

layout: col-sidebar
title: "K06: Overly Exposed Kubernetes Components"
---

## Overview

Exposing the Kubernetes components directly to the Internet leaks information about the cluster to attackers and also makes it simple for an attacker who has acquired a set of credentials for the component to use it, increasing the risk of cluster compromise.

## Description

Kubernetes presents a number of APIs which are used for cluster operations. When using the most popular managed Kubernetes distributions, the default is to expose the main Kubernetes API to the internet without any restrictions on source IP address. As a result of this there are well over a million Kubernetes API servers directly exposed to the Internet, as well as some Kubelet and etcd servers.

Directly exposing these APIs means that an attacker who is able to acquire a set of valid credentials (e.g. via a compromised administrator laptop or information leakage on GitHub) will be able to make use of them without restriction. 

It also presents an information leakage risk due to the way that those services work. The first risk is that, by default, Kubernetes exposes the exact version information via the `/version` endpoint, without authentication.

Additionally attackers can gain useful information via the TLS certificates used by the services, as they have SAN fields which are distinctive and commonly include information such as private network addresses.

Other components may also be exposed to the Internet, although this is a much less common configuration. Exposing the Kubelet or etcd could have serious consequences when they're combined with other configuration errors.

## How to prevent

Access to all Kubernetes APIs should be restricted to trusted hosts and networks. Most managed Kubernetes providers allow API server access to be restricted and this should be configured for all clusters.

For worker nodes, the Kubelet port should not be exposed to the Internet, access should only be allowed from the API server and any other specific hosts that require access.

## References

- [Kubernetes Census](https://raesene.github.io/blog/2024/02/17/a-final-kubernetes-censys/)  
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)  
- [Kubelet Authentication Docs](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)
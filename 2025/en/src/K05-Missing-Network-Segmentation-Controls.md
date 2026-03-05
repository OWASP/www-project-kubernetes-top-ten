---

layout: col-sidebar
title: "K05: Missing Network Segmentation Controls"
---

## Overview

By default Kubernetes does not restrict traffic between pods in the cluster or from pods to services in the control plane, which allows attackers to target those systems. This can present a risk especially in large multi-tenant clusters, where many services may be available.

## Description

Kubernetes provides the facility for any pod in a cluster to reach any other pod by default. This is an important aspect of simplifying deployment of applications to clusters, and providing inter-service connectivity.

The consequence of this flat network approach is that an attacker who is able to get access to a single pod in a cluster, will be able to probe any other application deployed to the network for weaknesses. Compounding this issue is that some applications may assume that the cluster network is "trusted" and therefore not implement robust authentication controls.

In addition to reaching other pods in the cluster, it's possible for an application in the cluster to contact administrative services running on the cluster nodes and the Kubernetes control plane. This can make it easier to exploit misconfigurations such as excessive RBAC privileges, where an application might be mistakenly given rights to a Kubernetes control plane API.

Where clusters lack outbound network policies, attackers who compromise a workload can easily exfiltrate data from the cluster, and also access services outside the cluster that might contain sensitive information (e.g. Cloud metadata services)

One specific configuration that can present risks to container network security is the use of host networking, which lets a pod in the cluster have access to a node's network interfaces. This allows that pod to contact localhost bound services on the node and also can exempt that pod from any network policy traffic restrictions that only apply to the container network.

CNI choice is also an important consideration as some CNIs (such as flannel) don't support network policies. In clusters using those CNIs network policies will be silently ignored, allowing unintended traffic.

## How to prevent

Kubernetes network policies are applied at a namespace level, so it's important that all namespaces in the cluster have network policies applied to them to restrict both inbound and outbound traffic. Network policies should start from a "default deny" approach and then allow traffic needed for the operation of the applications.

```yaml
# An example "default deny all" NetworkPolicy which prevents all ingress AND egress traffic for a specific namespace
# Note: this also denies traffic to the Kubernetes API Server and kube-system namespace (cluster dns resolution will not work unless appropriate NetworkPolicy is created)

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Host networking should only be used where explicitly necessary, for example services like the CNI pods themselves.

Depending on the CNI you're using, it may also be possible to apply network policies to Kubernetes hosts as well, and this can be used to improve the security of the cluster.

## References

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)  
- [Kubernetes Network Policy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)


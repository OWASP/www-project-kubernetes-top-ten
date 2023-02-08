## Overview
A Kubernetes cluster is compromised of many different components ranging from key-value storage within etcd, the kube-apiserver, the kubelet, and more. Each of these components are highly configurable have important security responsibilities. 

![Misconfigured Cluster Components - Illustration](/assets/images/K09-2022.gif)

## Description

Misconfigurations in core Kubernetes components can lead to complete cluster compromise or worse. In this section we will explore some of the components that exist on the Kubernetes control plane and nodes which can easily be misconfigured:

**kubelet:** Agent that runs on each node in the cluster and ensures that containers run as expected and are healthy. Some dangerous configurations to watch out for on the kubelet itself are as follows:

Anonymous authentication allows non-authenticated requests to the Kubelet. Check your Kubelet configuration and ensure the flag below is set to **false**:

```bash
#bad
--anonymous-auth=true
#good
--anonymous-auth=false
```

Authorization checks should always be performed when communicating with the Kubelets. It is possible to set the Authorization mode to explicitally allow unauthorized requests. Inspect the following to ensure this is not the case in your Kubelet config. The mode should be set to anything other than **AlwaysAllow**:

```bash
#bad
--authorization-mode=AlwaysAllow
#good
--authorization-mode=Webhook
```

**etcd:** A highly available key/value store that Kubernetes uses to centrally house all cluster data. It is important to keep etcd safe as it stores config data as well as secrets. 

**kube-apiserver:** The API server is a component of the Kubernetes [control plane](https://kubernetes.io/docs/reference/glossary/?all=true#term-control-plane)
 that exposes the Kubernetes API. The API server is the front end for the Kubernetes control plane. 

A simple security check you can perform is to inspect the internet accessibility of the API server itself. It is recommended to keep the Kubernetes API off of public network as seen in recent [news](https://www.bleepingcomputer.com/news/security/over-900-000-kubernetes-instances-found-exposed-online/). 

![Misconfigured Cluster Components - Mitigations](/assets/images/K09-2022-mitigation.gif)

## How to Prevent

A good start is to perform regular CIS Benchmark scans and audits focused on component misconfigurations. A strong culture of Infrastructure-as-Code can also help centralize Kubernetes configuration and remediation giving security teams visibility into how clusters are created and maintained. Using managed Kubernetes such as EKS or GKE can also help limit some of the options for component configuration as well as guide operators into more secure defaults. 

## References

CIS Benchmark: [https://www.cisecurity.org/benchmark/kubernetes](https://www.cisecurity.org/benchmark/kubernetes)

Kubernetes Cluster Components: [https://kubernetes.io/docs/concepts/overview/components/](https://kubernetes.io/docs/concepts/overview/components/)
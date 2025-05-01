---

layout: col-sidebar
title: "K01: Insecure Workload Configurations"
---

## Overview

The security context of a workload in Kubernetes is highly configurable which
can lead to serious security misconfigurations propagating across an
organization’s workloads and clusters. The [Kubernetes adoption, security, and
market trends report
2022](https://www.redhat.com/en/resources/kubernetes-adoption-security-market-trends-overview)
from Red Hat stated that nearly 53% of respondents have experienced a
misconfiguration incident in their Kubernetes environments in the last 12
months.

![Insecure Workload Configuration -
Illustration](../../../assets/images/K01-2022.gif)

## Description

Kubernetes manifests contain many different configurations that can affect the
reliability, security, and scalability of a given workload. These configurations
should be configured securely in-line with the recommendations of the 
[Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/). 

Some examples of high-impact manifest configurations are below:

**Application processes should not run as root:** Running the process inside of
a container as the `root` user is a common misconfiguration in many clusters.
While `root` may be an absolute requirement for some workloads, it should be
avoided when possible or, in Kubernetes 1.33 and above 
[user namespaces](https://kubernetes.io/blog/2025/04/25/userns-enabled-by-default/) 
can be used to provide root access inside the container while running as a 
non-privileged user from the host's perspective.
If the container were to be compromised, the attacker would have root-level 
privileges that allow actions such as starting a malicious
process that otherwise wouldn’t be permitted with other users on the system.

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: root-user
spec:  
  containers:
  ...
  securityContext:  
    #root user:
    runAsUser: 0
    #non-root user:
    runAsUser: 5554
```

**Read-only filesystems should be used:** In order to limit the impact of a
compromised container on a Kubernetes node, it is recommended to utilize
read-only filesystems when possible. This prevents a malicious process or
application from writing back to the host system. Read-only filesystems are a
key component to preventing container breakout.

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: read-only-fs
spec:  
  containers:  
  ...
  securityContext:  
    #read-only fs explicitly defined
    readOnlyRootFilesystem: true
```

**Privileged containers should be disallowed**: When setting a container to
`privileged` within Kubernetes, the container can access additional resources
and kernel capabilities of the host. Workloads running as root combined with
privileged containers can be devastating as the user can get complete access to
the host. Privileged containers are dangerous as they remove many of the built-in
container isolation mechanisms entirely.

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: privileged-pod
spec:  
  containers:  
  ...
  securityContext:  
    #priviliged 
    privileged: true
    #non-privileged 
    privileged: false
```

**Resource constraints should be enforced**: By default, containers run with
unbounded compute resources on a Kubernetes cluster. CPU requests and limits
can be attributed to individual containers within a pod. If you don't specify
a CPU limit for a container, it means there's no upper bound on the CPU
resources it can consume. While this flexibility can be advantageous, it also
poses a risk for potential resource abuse, such as crypto-mining, as the
container could potentially utilize all available CPU resources on the
hosting node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-limit-pod
spec:
  containers:
  ...
    resources:
      limits:
        cpu: "0.5" # 0.5 CPU cores
        memory: "512Mi" # 512 Megabytes of memory
      requests:
        cpu: "0.2" # 0.2 CPU cores
        memory: "256Mi" # 256 Megabytes of memory
```

## How to Prevent

Preventing users from running workloads with excessive privileges can be achieved 
by using admission control services which will enforce policies on workloads 
in the cluster. There are a range of available options to achieve this, both built-in
to the Kubernetes project and also via third party applications.

Within the Kubernetes project there are two available options. 
[Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
is a relatively simple system that allows for one of three security levels to be defined for
each namespace in the cluster. It is suitable for simple use cases but doesn't suit more complex
requirements.

[Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
provides more fine grained controls on workloads in Kubernetes clusters. Policies written in 
[Common Expression Language (CEL)](https://github.com/google/cel-spec) can be used to restrict
workload configuration in the cluster.

There are also external admission control options available which can be used to control the 
security of workloads in the cluster. [Kyverno](https://kyverno.io/) and [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)
are two popular projects in this field. Both provide flexible policy enforcement options.

![Insecure Workload Configuration -
Mitigations](../../../assets/images/K01-2022-mitigation.gif)

## Example Attack Scenarios

TODO

## References

CIS Benchmarks for Kubernetes:
[https://www.cisecurity.org/benchmark/kubernetes](https://www.cisecurity.org/benchmark/kubernetes)

Pod Security Standards:
[https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

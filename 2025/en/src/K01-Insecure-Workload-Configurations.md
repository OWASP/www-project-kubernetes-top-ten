---

layout: col-sidebar
title: "K01: Insecure Workload Configurations"
---

## Overview

By default, workloads running in a Kubernetes cluster do not have strong security posture, which could make it easier for attackers to compromise applications and expand their compromise to other workloads or the cluster control plane.

It's vital that `securityContext` settings are implemented and other hardening steps are applied to improve the overall security of the cluster.

## Description

By default Kubernetes and the underlying container runtimes provide simple defaults which make it easy for application developers to run their workloads in the cluster. However in a number of places these defaults are not hardened for production deployment. Below are some of the key places where default values need to be changed:

- **Running as root**. Running containers as the root user (UID 0) is dangerous as that user has special privileges on a Linux host, including access to a wider set of kernel code paths, increasing the risk of container breakout.
- **Default Linux capabilities**. Container runtimes will provide a set of Linux capabilities which are portions of the overall root privileges to all containers. Some of these capabilities can be dangerous and make it easier for attackers to attempt to compromise other containers or systems in the cluster.
- **Service Account Tokens**. Each Pod gets a service account token mounted by default which provides access to the Kubernetes API. This can allow an attacker to gain more information about the cluster and, if mis-configured, could allow for compromise of the cluster.
- **Removing Seccomp Filters**. Major container runtimes like containerd add a seccomp filter which helps to reduce the risk of container breakout, however Kubernetes defaults to Unconfined seccomp mode, meaning the runtime's built-in filter is not applied unless explicitly configured.
- **Missing Resource Limits**. Kubernetes does not enforce CPU/memory limits by default. A compromised container can exhaust node resources, causing Denial of Service for co-located workloads.

In addition to dangerous defaults there are some workload security settings which should be avoided wherever possible.

- **Privileged**. A privileged container will be able to break out to the underlying host, and this setting should not be used unless absolutely required.
- **Host namespaces**. Containers use namespaces to provide isolation from the underlying host, and removing that protection weakens container isolation. Workloads should not use host namespaces unless their functionality explicitly requires it.

## How to prevent

There are a number of ways in which the configuration of Kubernetes workloads can be improved from the default values. The first place to look is in the security context section, but some of the settings are also in the general pod or container specification.

There are a large number of available options which can be set at either the pod level or at the level of the individual container. Whilst all of them can be important, we'll discuss some key options that every workload should consider setting as they make major improvements to the security of workloads running in clusters.

### Running as non-root

One of the most important changes in workload security is to ensure that your applications 
do not run as the root user on the host. Running as the root user makes it much easier for an attacker who compromises an application running in a container to break out to the underlying cluster node.

There are a couple of different approaches that can be used to achieve this. Firstly you can specify the `runAsUser` and `runAsGroup` for all containers in a pod. In the example all processes in containers in this pod will run as UID 1000 and GID 3000.

```yaml
 securityContext:
    runAsUser: 1000
    runAsGroup: 3000
```

Another approach, enabled by default since Kubernetes 1.33 and stable since 1.36, is to use user namespaces to ensure that your containers won't run as root on the host, even if they are running in root in the container. This can be specified with the `hostUsers` setting. The nodes need a container runtime and kernel that support it, see [User Namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/).

```yaml
spec:
  hostUsers: false
```


### Dropping Capabilities

By default containers get given a set of capabilities for privileged operation. Removing these capabilities reduces the risk of container breakout and most applications won't need them.

```yaml
      securityContext:
        allowPrivilegeEscalation: false
        privileged: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

### Enable readOnlyRootFilesystem

Most malware and exploit payloads require writing to /tmp or /bin. Forcing a read-only root FS makes it significantly harder for an attacker to maintain persistence or install tools. If applications need to write logs or temp files should use `emptyDir` volumes instead of writing to the container layer.

### Disable allowPrivilegeEscalation

The ‍`allowPrivilegeEscalation: false` setting enforces the `no_new_privs` kernel flag, which prevents a process from gaining more privileges than its parent. In a standard Linux environment, an attacker can exploit files with the SUID bit to temporarily elevate their effective UID to root, even if the container started as a non privileged user. By disabling this escalation, you ensure that the kernel blocks any attempts to bypass user restrictions, effectively neutralizing a primary path for local privilege escalation and reducing the risk of a full container breakout

### Don't mount service account tokens

Unless your application needs to communicate with the Kubernetes API, you should configure the pod not to mount a service account token.

```yaml
    spec:
      automountServiceAccountToken: false
```

### Enable Seccomp Filter

Re-enabling the container runtime's seccomp filter will improve the overall security of the workload and reduce the risk of container breakout.

```yaml
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

Cluster operators can also make this the default for every Pod on a node by running the kubelet with `--seccomp-default` (or `seccompDefault: true` in the kubelet configuration), which is stable since Kubernetes 1.27. Pods that set no profile then get `RuntimeDefault` instead of `Unconfined`. Roll it out gradually, since a workload that relies on a blocked syscall will fail at runtime; see [Enable the use of RuntimeDefault as the default seccomp profile](https://kubernetes.io/docs/tutorials/security/seccomp/#enable-the-use-of-runtimedefault-as-the-default-seccomp-profile-for-all-workloads).

### Set resource requests and limits

Give each container CPU and memory requests and limits so a compromised or misbehaving workload can't starve the rest of the node. A `LimitRange` in each namespace applies defaults to containers that don't set their own, and a `ResourceQuota` caps the total a namespace can use.

```yaml
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          memory: 256Mi
```


## References

 - [Kubernetes Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
 - [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
 - [CIS Benchmark for Kubernetes](https://www.cisecurity.org/benchmark/kubernetes) - Sections 5.1 and 5.2
 - [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
 - [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
 

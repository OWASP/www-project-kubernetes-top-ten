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
from Redhat stated that nearly 53% of respondents have experienced a
misconfiguration incident in their Kubernetes environments in the last 12
months.

![Insecure Workload Configuration -
Illustration](../../../assets/images/K01-2022.gif)

## Description

Kubernetes manifests contain many different configurations that can affect the
reliability, security, and scalability of a given workload. These configurations
should be audited and remediated continuously. Some examples of high-impact
manifest configurations are below:

**Application processes should not run as root:** Running the process inside of
a container as the `root` user is a common misconfiguration in many clusters.
While `root` may be an absolute requirement for some workloads, it should be
avoided when possible. If the container were to be compromised, the attacker
would have root-level privileges that allow actions such as starting a malicious
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
the host. This is, however, limited when running as a non-root user. Privileged
containers are dangerous as they remove many of the built-in container isolation
mechanisms entirely.

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

Maintaining secure configurations throughout a large, distributed Kubernetes
environment can be a difficult task. While many security configurations are
often set in the `securityContext` of the manifest itself there are a number of
other misconfigurations that can be detected elsewhere. In order to prevent
misconfigurations, they must first be detected in both runtime and in code. We
can enforce that applications:

1. Run as non-root user
2. Run as non-privileged mode
3. Set AllowPrivilegeEscalation: False to disallow child process from
getting more privileges than its parents.
4. Set a LimitRange to constrain the resource allocations for each applicable
object kind in a namespace.

Tools such as Open Policy Agent can be used as a policy engine to detect these
common misconfigurations. The CIS Benchmark for Kubernetes can also be used as a
starting point for discovering misconfigurations.

![Insecure Workload Configuration -
Mitigations](../../../assets/images/K01-2022-mitigation.gif)

## Example Attack Scenarios

Example #1: Testing Network Vulnerabilities From Within A Pod

Even if you're inside of the network (internal LAN, externally, etc.), an attacker may not have a machine to test for vulnerabilities on the cluster network. They could use a Pod that has root permissions to bring down certain packages and scan the environment, like Nmap.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginxdeployment
  replicas: 1
  template:
    metadata:
      labels:
        app: nginxdeployment
    spec:
      containers:
      - name: nginxdeployment
        image: nginx:latest
        securityContext:
          allowPrivilegeEscalation: true
          privileged: true
        ports:
        - containerPort: 80
```

```
kubectl exec -ti nginx-deployment-6577b4688f-w9sw9 -- bash
```

```
apt install nmap -y
```

You may be thinking "but the Pod is on it's own network, so how could it see the host network?". Even though that's true, it's still within the host network. For example, if you run an `ifconfig` in the Pod, you’ll see the Pods IP address that was handed out from the CNI.

```
root@nginx-deployment-6577b4688f-w9sw9:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.81  netmask 255.255.255.255  broadcast 0.0.0.0
```

However, if you run an `nmap` scan on the IP range that the Control Plane and Worker Nodes are on, you'll see every host within that network, including the Control Plane (kubernetes.default.svc.cluster.local)
```
root@nginx-deployment-6577b4688f-w9sw9:/# nmap -sn 192.168.1.0/24 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-08-23 19:17 UTC
Nmap scan report for Gateway (192.168.1.1)
Host is up (0.0083s latency).
Nmap scan report for 192.168.1.13
Host is up (0.063s latency).
Nmap scan report for talos-2vg-tzm (192.168.1.31)
Host is up (0.00034s latency).
Nmap scan report for ubuntu-server (192.168.1.49)
Host is up (0.00039s latency).
Nmap scan report for 192.168.1.70
Host is up (0.00019s latency).
Nmap scan report for 192-168-1-76.hubble-peer.kube-system.svc.cluster.local (192.168.1.76)
Host is up (0.000063s latency).
Nmap scan report for 192-168-1-100.kubernetes.default.svc.cluster.local (192.168.1.100)
Host is up (0.00026s latency).
Nmap scan report for talos-74k-uwa (192.168.1.110)
Host is up (0.00066s latency).
```

Now that you know the Control Plane IP address, you can scan what Ports are open.

```
nmap --top-ports 10 192.168.1.100
Starting Nmap 7.93 ( https://nmap.org ) at 2024-08-23 19:22 UTC
Nmap scan report for 192-168-1-100.kubernetes.default.svc.cluster.local (192.168.1.100)
Host is up (0.00026s latency).

PORT     STATE  SERVICE
21/tcp   closed ftp
22/tcp   open   ssh
23/tcp   closed telnet
25/tcp   closed smtp
80/tcp   closed http
110/tcp  closed pop3
139/tcp  closed netbios-ssn
443/tcp  closed https
445/tcp  closed microsoft-ds
3389/tcp closed ms-wbt-server
```



## References

CIS Benchmarks for Kubernetes:
[https://www.cisecurity.org/benchmark/kubernetes](https://www.cisecurity.org/benchmark/kubernetes)

Open Policy Agent:
[https://github.com/open-policy-agent/opa](https://github.com/open-policy-agent/opa)

Pod Security Standards:
[https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

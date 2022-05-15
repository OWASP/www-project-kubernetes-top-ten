## Overview

The security context of a workload in Kubernetes is highly configurable which can lead to serious security misconfigurations propagating across and organization’s workloads and clusters. The 2021 Kubernetes Security Survey from Redhat stated that nearly 60% of respondents have experienced a misconfiguration incident in their Kubernetes environments in the last 12 months.

## Description

Kubernetes manifests contain many different configurations that can effect the reliability, security, and scalability of a given workload. These configurations should be audited and remediated continuously. Some examples of high-impact manifest configurations are below:

**Application processes should not run as root:** 

**Read-only filesystems should be used:**

**Privileged containers should be disallowed**: Setting a container to `privileged` within Kubernetes, the container itself can access additional resources and kernel capabilities of the host itself. Privileged containers are dangerous as they remove many of the built-in container isolation mechanisms entirely. 

**The host network and process space should not be used:**

## How to Prevent

Maintaining secure configurations throughout a large, distributed Kubernetes environment can be a difficult task. Security configurations are typically set in the security context of the manifest itself. See below for an example: 

## Example Attack Scenarios

Example #1: Container Breakout 1-Liner

Example #2: Docker Socket Mount

## References

https://www.cisecurity.org/benchmark/kubernetes

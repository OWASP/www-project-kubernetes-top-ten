---

layout: col-sidebar
title: "K08: Cluster-To-Cloud Lateral Movement"
---

## Overview

When Kubernetes is run on a cloud provider, the cluster's nodes and workloads are often granted permissions to interact with other cloud services (e.g., object storage, databases, metadata services). If an attacker compromises a container within the cluster, they can attempt to abuse these cloud permissions to "move laterally" from the Kubernetes cluster into the underlying cloud provider account, leading to a much wider and more severe breach.

Securing the boundary between the Kubernetes cluster and the cloud provider's API is essential to contain the impact of a potential compromise.

## Description

The connection between a Kubernetes cluster and its host cloud environment is usually established through a cloud-specific Identity and Access Management (IAM) mechanism. The primary ways this access is granted and can be abused include:

- **Overly Permissive Node Roles/Policies**: In many default configurations, the Kubernetes worker nodes are assigned a single, broad IAM role. Every pod running on a given node can, by default, inherit these powerful permissions. An attacker who gains execution in any container on that node can potentially access the node's credentials and use them to attack other cloud resources.  
- **Static Cloud Credentials**: A common anti-pattern is storing static, long-lived cloud provider credentials (e.g., access keys, service principal secrets) directly in Kubernetes Secrets, ConfigMaps, or embedding them in container images. These static credentials are a prime target for attackers, as they are often long-lived and, if compromised, provide persistent access to the cloud account. See K03 for more information on secrets management.  
- **Unrestricted Access to Instance Metadata**: Cloud providers expose a metadata service on a special IP address (e.g., 169.254.169.254) that is reachable from the worker nodes. This service provides information about the instance, including the temporary credentials associated with its IAM role. By default, pods can often access this metadata service, allowing them to steal the node's credentials.

An attacker exploiting these weaknesses could perform actions like exfiltrating data from S3 buckets or RDS databases, creating new virtual machines, or disrupting cloud infrastructure, all from a single compromised pod.

## How to prevent

The core principle for preventing cluster-to-cloud lateral movement is following the principle of least privilege, ensuring that pods only have the precise, minimal permissions they need to the specific cloud services they must access.

### Use Workload-Specific Cloud Identities

The most effective, modern approach is to associate cloud IAM roles directly with Kubernetes ServiceAccounts, rather than with the underlying nodes. This practice is supported by all major cloud providers and ensures that a pod's cloud permissions are independent of the node it is running on.

- Amazon EKS: [*EKS Pod Identity*](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) or [IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html).  
- Google Kubernetes Engine (GKE): Called [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity).  
- Azure Kubernetes Service (AKS): Called [Workload Identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview).

This mechanism allows a cluster administrator to create a fine-grained IAM role in the cloud provider and bind it to a specific ServiceAccount in Kubernetes. Pods that use this ServiceAccount are then able to automatically retrieve short-lived, scoped credentials for that role only.

Example: Kubernetes ServiceAccount with AWS IRSA Annotation

This ServiceAccount is annotated to request the permissions defined in the specified AWS IAM role.

```yaml
# This annotation is specific to Amazon EKS (IRSA)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-app
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/MyApplicationRole"
```

### Block Access to Node Metadata Service

To provide defense-in-depth, you should block pods from accessing the underlying node's metadata service. This prevents a compromised pod from being able to steal the node's broader IAM credentials. This can be achieved with a Kubernetes NetworkPolicy.

This policy denies all pods in the namespace from sending traffic to the standard cloud provider metadata IP address (N.B. Some cloud providers have additional metadata IP addresses).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata-access
  namespace: default # Apply to the desired namespace
spec:
  podSelector: {} # Applies to all pods in this namespace
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
```



### Avoid Static Credentials

Never store static, long-lived cloud credentials in your cluster (or in any infrastructure\!). Always prefer using short-lived, automatically rotated credentials that will eliminate the risk of a leaked static secret.

### Implementing behavioral monitoring and runtime security

Lateral movement typically begins with the execution of unauthorized tools (curl, nmap, or aws-cli) or the spawning of unexpected shells within a compromised container. Implementing runtime security tools (such as Falco or Tetragon) allows for continuous monitoring of container behavior and instant alerting on suspicious activity. By detecting anomalous process execution an attack can be intercepted in its early stages.

### Harden Metadata services

Where possible look at cloud provider settings which can improve the security of instance metadata services. For example in AWS IMDSv2 should be implemented on all hosts.

## References

- [AWS EKS: IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)  
- [Google GKE: Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)  
- [Azure AKS: Microsoft Entra Workload ID](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)  
- [Kubernetes Docs: Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
---

layout: col-sidebar
title: "K02: Overly Permissive Authorization Configurations"
---

## Overview

Kubernetes provides a rich and flexible set of options for authorizing users and services to operate on cluster objects. With that flexibility, goes the risk of excessive access which could allow for unauthorized access to Kubernetes resources.

## Description

There are a number of things to think about when looking at authorization configuration in Kubernetes.

### Common RBAC Mis-configurations

There are a number of places where RBAC misconfigurations commonly occur. For more detailed guidance see the [Kubernetes Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)

- **Using cluster-admin.** The cluster-admin clusterrole in Kubernetes provides unlimited access to all cluster resources, including sensitive data such as secrets. Allowing users to use this role also presents a risk of accidental deletion of modification of resources in the cluster.  
- **Using in-built Kubernetes Clusterroles.** Kubernetes provides a number of in-built clusterroles such as `view` and `edit`. It can be tempting to use these options as they are built-in, however they can provide quite wide ranging permissions that don't align with the user's needs.  
- **LIST & WATCH rights to secrets.** Whilst it's clear that allowing users to GET secret objects would allow for their disclosure, it's also important to note that allowing LIST or WATCH rights to secrets also allows users to see the contents of those secrets.  
- **Node/proxy rights provide Kubelet API access.** Any user who gets rights to the proxy sub-resource of node objects in Kubernetes can connect to the Kubelet API directly bypassing audit logging and admission control requirements, so care should be taken when granting that access.  
- **Using ESCALATE, BIND and IMPERSONATE permissions.** Using the verbs like escalate, bind and impersonate can allow users to bypass protections of privilege escalation. The escalate verb allows users to create a clusterrole with more rights than they already have. The bind permission allows users to bind to a clusterrole to which they are not already bound. The impersonate user allows users to impersonate other users in the cluster.  
- **PATCH on namespace.** Users who have patch permissions to namespaces can update the namespace labels. In clusters where Pod Security Admission is used, this may allow a user to configure the namespace for a more permissive policy than intended by the administrators. For clusters where NetworkPolicy is used, users may be set labels that indirectly allow access to services that an administrator did not intend to allow.  
- **Third Party Shadow RBAC.** A significant yet frequently overlooked risk is the introduction of "Shadow RBAC" through the deployment of third party Helm charts, Operators, or external manifests. To ensure out of the box functionality, many community maintained packages default to over privileged configurations, such as the silent provisioning of ClusterRole/ClusterRoleBinding and  the excessive use of wildcards `*` for API verbs and resources. This creates an expansive blast radius where a compromised application inherits broad discovery or administrative capabilities across the entire cluster. Furthermore many charts fail to disable the automatic mounting of ServiceAccount tokens, providing an immediate vector for lateral movement. An attacker gaining shell access can leverage these pre authenticated credentials to query the API server, even if the application itself has no functional requirement.  
- **Be aware of ClusterRole Aggregation.** Third-party operators can silently extend built-in roles (like admin) by adding matching labels to their custom ClusterRoles, effectively granting themselves admin rights.

 
### Multiple authorizers

Kubernetes allows for multiple different authorization modes to exist in every cluster. In most clusters RBAC will be used for authorization, however Webhook authorization is also commonly used in managed Kubernetes clusters.

The rights that a user or service has to the cluster will be the union of all rights provided from any configured authorization mechanism, so it's important to check all configured options when reviewing a cluster's authorization posture.

Another important point to note is that in-built Kubernetes features like `kubectl auth can-i --list` do not take account of rights provided by non-RBAC authorization systems, so effective auditing of Kubernetes permissions must be done for each authorizer in the cluster.

## How to prevent

The main principle that should be followed when designing authorization in Kubernetes clusters is to use a "least privileges" approach. Excessive general privileges should be avoided both for interactive users and services accessing the Kubernetes API.

For administrators, consider carefully using the impersonation system in Kubernetes to provide a lower privileged main account which can escalate its privileges when needed for specific operations as this will help avoid accidental damage to Kubernetes objects. Note that the impersonation system should be used with care only allowing escalation to specific resource names and not to all users.

In as much as possible, rights should be provided through a single authorization method and not via a mix of different authorizers as it is difficult to audit Kubernetes permissions across multiple authorization systems.

While the principle of least privilege must be the baseline for all kubernetes authorization, operational cases such as emergency debugging of a production incident or performing a sensitive one off change necessitate temporary elevated access. To manage these risks , organizations(the cluster admin) should move beyond the static assignment toward a process driven architecture. Granting privileged access should never be an adhoc manual task. It must be a process oriented workflow where every change is documented. In addition to Kubernetes audit logs, the access grant and related changes should be recorded and traceable outside the cluster as well, typically through GitOps (e.g.,. pull requests and an immutable change history).

For a more resilient security posture and to reduce clusters attack surface, organizations should implement Just In Time (JIT) Access, with tools that create and expire ClusterRoleBindings and RoleBindings on demand. Simple implementations can use short-lived kubeconfig files with OIDC token expiry. This approach effectively eliminates standing privileges by providing ephemeral on demand access. JIT access replaces broad permanent privileges with dynamic time bound authorizations that automatically expire after a specific time. By leveraging granular approval workflows such as multi party verification or context aware auto approvals, JIT ensures that elevated rights are precisely aligned with actual operational needs. This dynamic approach provides a clear, comprehensive audit trail, making it easier to document exactly who had access to what, when, and why.

## References

- [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)  
- [CIS Benchmark for Kubernetes](https://www.cisecurity.org/benchmark/kubernetes) - Section 5.1
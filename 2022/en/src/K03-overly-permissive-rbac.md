## Overview
[Role-Based Access Control](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (RBAC) is the primary authorization mechanism in Kubernetes and is responsible for permissions over resources. These permissions combine verbs (get, create, delete, etc.) with resources (pods, services, nodes, etc.) and can be namespace or cluster scoped. A set of out of the box roles are provided that offer reasonable default separation of responsibility depending on what actions a client might want to perform. Configuring RBAC with least privilege enforcement is a challenge for reasons we will explore below.

## Description
RBAC is an extremely powerful security enforcement mechanism in Kubernetes when appropriately configured but can quickly become a massive risk to the cluster and increase the blast radius in the event of a compromise.  Below are a few examples of misconfigured RBAC:

*Unnecessary use of `cluster-admin`*

When a subject such as a Service Account, User, or Group has access to the built-in Kubernetes “superuser” called `cluster-admin` they are able to perform any action on any resource within a cluster. This level of permission is especially dangerous when used in a `ClusterRoleBinding` which grants full control over every resource across the entire cluster. `cluster-admin` can also be used as a `RoleBinding` which may also pose significant risk.

Below you will find the RBAC configuration of a popular OSS Kubernetes development platform. It showcases a very dangerous `ClusterRoleBinding` which is bound to the `default` service account. Why is this dangerous? It grants the all-powerful `cluster-admin` privilege to every single Pod in the `default` namespace. If a pod in the default namespace is compromised (think, Remote Code Execution) then it is trivial for the attacker to compromise the entire cluster by impersonating the service 

```jsx

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
 name: redacted-rbac
subjects:
 - kind: ServiceAccount
   name: default
   namespace: default
roleRef:
 kind: ClusterRole
 name: cluster-admin
 apiGroup: rbac.authorization.k8s.io
```

## How to Prevent

To reduce the risk of an attacker abusing RBAC configurations, it is important to analyze your configurations continuously and ensure the principle of lease privilege is always enforced. Some recommendations are below:

- Reduce direct cluster access by end users when possible
- Don’t use Service Account Tokens outside of the cluster
- Avoid automatically mounting the default service account token
- Audit RBAC included with installed third-party components
- Deploy centralized polices to detect and block risky RBAC permissions
- Utilize `RoleBindings` to limit scope of permissions to particular namespaces vs. cluster-wide RBAC policies

## Example Attack Scenarios
An OSS cluster observability tool is installed inside of a private Kubernetes cluster by the platform engineering team. This tool has an included web UI for debugging and analyzing traffic. The UI is accidentally exposed to the internet through it’s included Service manifest - it uses type: LoadBalancer which spins up an AWS ALB load balancer with a **public** IP address. 

This hypothetical tool uses the following RBAC configuration:

```jsx
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-sa-namespace-admin
  namespace: prd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: system:serviceaccount:prd:default
```

An attacker finds the open web UI and is able to get a shell on the running container in the cluster. The default service account token in the `prd` namespace is used by the web UI and the attacker is able to impersonate it to call the Kubernetes API and perform elevated actions such as `describe secrets` in the `kube-system` namespace. This is due to the `roleRef` which gives that service account the built-in privilege `admin` in the entire cluster. 

## References

Kubernetes RBAC: [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

[https://github.com/PaloAltoNetworks/rbac-police](https://github.com/PaloAltoNetworks/rbac-police)
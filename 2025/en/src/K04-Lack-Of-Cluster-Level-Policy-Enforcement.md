---

layout: col-sidebar
title: "K04: Lack Of Cluster Level Policy Enforcement"
---

## Overview

Out of the box, Kubernetes provides significant flexibility in how workloads and resources are configured. While this is powerful, it lacks a default, cluster-wide mechanism for enforcing security policies. This can lead to security gaps and misconfigurations as clusters scale and are managed by multiple teams.

It is important to leverage Kubernetes' built-in policy mechanisms and, where necessary, its native extension points to ensure that all resources deployed to the cluster adhere to organizational and security best practices.

## Description

Without a centralized policy enforcement strategy, organizations and individuals often rely on manual reviews, CI/D pipelines, or custom review scripts. These approaches are difficult to scale, prone to human error, and often cannot prevent a user with direct cluster access from creating non-compliant resources. This can lead to several security risks:

- **Inconsistent Security Posture**: Different teams may configure their workloads with varying levels of security, leading to exploitable weak points in the cluster.  
- **Insecure Configurations**: Users with sufficient permissions might inadvertently create resources that violate security best practices, such aos running privileged pods, exposing services publicly, or using container images from untrusted registries.  
- **Unrestricted Network Paths**: Without defined controls, all pods in a cluster can communicate with each other by default, potentially allowing a compromised application to move laterally and access sensitive services.  
- **Compliance and Governance Challenges**: For organizations in regulated industries, demonstrating compliance is difficult without a system that can audit and enforce policies across all environments in a verifiable way.  
- ​​**Metadata Trust Abuse**: Kubernetes components and addons rely on labels/annotations for routing and security decisions. Without enforced validation, users can spoof metadata to bypass controls and gain unauthorized access, for example modifying namespace labels to bypass Pod Security Admission checks.

Kubernetes provides several built-in resources and controllers to address these issues, which should be the first line of defense in any policy enforcement strategy. We can use these policies to prevent the misconfigurations that so often lead to compromises in Kubernetes workloads.

## How to prevent

The primary method for enforcing policies within Kubernetes is through Admission Controllers. An admission controller is a piece of code that intercepts requests to the Kubernetes API server before an object is created or modified. As detailed in the [official documentation](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/), this allows for the validation or mutation of resources before they are stored.

### Built-in Policy Enforcement

Kubernetes offers several powerful, built-in mechanisms for policy enforcement that should be implemented first.

#### Pod Security Admission

The easiest to implement of these is Pod Security Admission (PSA), which can be configured, at a namespace level, to restrict workloads in-line with the levels defined by Kubernetes Pod Security Standards. It's also possible to use PSA to gather information about what workloads are using privileges by only auditing failures rather than blocking.

Restrictions are added using labels on Kubernetes namespaces, as shown in the example below.

```yaml
# automatically reject any new pods in this namespace that do not meet
# the 'baseline' policy requirements.

apiVersion: v1
kind: Namespace
metadata:
  name: my-secure-app
  labels:
    # ENFORCE: Any pod that violates the baseline policy will be rejected.
    pod-security.kubernetes.io/enforce: baseline
    
    # AUDIT: Pods violating the baseline policy will be allowed, but an
    # audit event will be recorded. This is useful for tracking compliance       
    # without breaking workloads.
    pod-security.kubernetes.io/audit: baseline
    
    # WARN: A user-facing warning will be displayed to the user when they
    # create a pod that violates the baseline policy.
    pod-security.kubernetes.io/warn: baseline

```

#### Validating Admission Policy

Kubernetes also provides a more flexible option using Validating Admission Policy (VAP). This allows cluster administrators to define policies using Common Expression Language (CEL).

This option is more flexible than PSA as it allows for specific policies to be designed and implemented, depending on the cluster's threat model. It also doesn't require any additional software to be deployed to support it (as external admission webhooks do)

### Admission Webhooks

For policies that go beyond what PSA or VAP can offer, Kubernetes provides a native extension mechanism: Dynamic Admission Controllers. These are configured as ValidatingAdmissionWebhook or MutatingAdmissionWebhook resources.

These webhooks allow you to deploy your own custom policy logic, which the Kubernetes API server will call before admitting a resource. While you can build these webhooks from scratch, the community has developed several open-source policy engines that leverage this mechanism to provide powerful, flexible policy management.

When considering such a tool, ensure it is built upon the native Kubernetes webhook interface. Popular examples include:

- OPA/Gatekeeper: Uses the Open Policy Agent engine and a Kubernetes-native integration (Gatekeeper) to enforce policies written in the Rego language.  
- Kyverno: A policy engine designed specifically for Kubernetes that allows you to write policies as simple Kubernetes YAML resources.  
- Kubewarden: A policy engine that allows you to write policies in your preferred programming language which are compiled into WebAssembly modules and stored in container registries

These tools can enforce a wide range of custom policies, such as:

- Requiring specific labels on all resources.  
- Restricting which container registry images can be pulled from.  
- Enforcing specific Ingress configurations.

When using these tools, you are building upon the extensibility provided by Kubernetes itself, allowing you to create a comprehensive, layered security strategy. To complete this strategy, centralizing the admission controller logs into a Security Information and Event Management (SIEM) is essential for real time threat detection and auditing.

### Safe Rollouts

Regardless of the admission control solution you choose, always start with audit or warn modes to identify violations without risking operational disruption. For example, with webhooks, use failurePolicy: Ignore during initial rollout to prevent cluster lockouts if the policy engine fails.

## References

- [Kubernetes Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)  
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)  
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)  
- [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)  
- [Common Expression Language](https://cel.dev/)  
- [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/)  
- [Kyverno](https://kyverno.io/)  
- [Kubewarden](https://www.kubewarden.io/)


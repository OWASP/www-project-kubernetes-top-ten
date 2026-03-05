---

layout: col-sidebar
title: "K09: Broken Authentication Mechanisms"
---

## Overview

Authentication to the Kubernetes API—and more broadly access control for cluster users and workloads—can take many forms (certificates, tokens, OIDC, etc.). This flexibility helps adapt to different environments, but also opens the door to misconfigurations or weak authentication setups. A “broken authentication” posture significantly increases the risk of unauthorized access, privilege escalation, and cluster compromise.

## Description

Several entities require access to the Kubernetes API, with authentication being the first gate each request must pass. Kubernetes API authentication is performed over HTTP, and the specific authentication mechanism can vary by cluster configuration. Requests that fail authentication are denied and return an HTTP 401 (Unauthorized) response.

Kubernetes provides several in-built authentication mechanisms for users, however they are not suitable for production cluster use due to lacking security features such as user revocation or auditing. For example, client certificates issued by Kubernetes clusters cannot easily be revoked and are often valid for years, presenting considerable risks of credential theft.

## How to prevent

### Use Strong User Authentication

Preventive concept: Enforce robust authentication for all API access using secure, externally managed identity providers instead of weak or static credentials.

Kubernetes supports external methods like OIDC and JWT authenticators for production, issuing cryptographically signed tokens that integrate with enterprise IdPs and short‑lived credentials. Structured Authentication allows multiple JWT authenticators and flexible claim validation, making it easier to enforce strong, policy-driven authentication.

### Avoid using certificates for end user authentication

Certificates are convenient for authenticating to the Kubernetes API but should be used with extreme caution. At this time, the API has no mechanism to revoke certificates, which can lead to a scramble to re-key the cluster in the event of a compromise or leakage of private key material. Certificates are also more cumbersome to configure, sign, and distribute. They may be appropriate as a “break glass” authentication mechanism, but not for primary authentication.

### Never roll your own authentication
As with cryptography, you should avoid building novel authentication mechanisms when they are not necessary. Use authentication methods that are supported and widely adopted.

### Enforce MFA when possible

Regardless of the authentication mechanism chosen, require humans to provide a second factor of authentication, typically enforced through an OIDC provider.

### Don’t use Service Account tokens from outside of the cluster

For use inside the cluster, Kubernetes Service Account tokens are obtained directly via the TokenRequest API and mounted into Pods using a projected volume. For use outside the cluster, these tokens must be manually provisioned through a Kubernetes Secret and typically have no expiration. Using long-lived Service Account tokens from outside the cluster introduces significant risk.

If a token-based approach is required, short-lived tokens can be provisioned using the TokenRequest API or via kubectl create token with the `--duration` flag.

### Authenticate users and external services using short-lived tokens

All authentication tokens should be as short-lived as tolerable. This ensures that if (and when) a credential is leaked, it may expire before it can be replayed to compromise the account.

### Disable anonymous authentication on Kubernetes APIs

Kubernetes supports anonymous API access by default, assigning unauthenticated requests the system:anonymous identity. If authorization is misconfigured, this can expose API endpoints without explicit authentication.

For self-managed clusters, disable anonymous access by setting `--anonymous-auth=false` on the API server and Kubelet. For managed Kubernetes services, ensure no RBAC permissions are granted to the system:unauthenticated group.

Disabling anonymous authentication ensures all Kubernetes API requests require an explicit identity, reducing the risk of authentication bypass, although this can have availability consequences, so cluster operators should take care to test this change.

### Don’t Use Default Service Account Tokens

For use inside the cluster, Pods are automatically assigned the default service account if no explicit service account is specified. This default token is mounted into Pods automatically and can grant unintended API access. Using default service account tokens for workloads opens your cluster up to significant risk.

Where API access is required, create explicit service accounts for each workload and disable automatic mounting of the default token with `automountServiceAccountToken: false`. This ensures that only intended Pods have API credentials.

## References

- [Kubernetes Documentation - Controlling Access](https://kubernetes.io/docs/concepts/security/controlling-access/)  
- [Kubernetes Documentation - Authentication](https://kubernetes.io/docs/concepts/security/hardening-guide/authentication-mechanisms/)   
- [Kubernetes Documentation - Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)  
- [Kubernetes Structured Authentication](https://kubernetes.io/blog/2024/04/25/structured-authentication-moves-to-beta/)


---

layout: col-sidebar
title: "K03: Secrets Management Failures"
---

## Overview

Applications running in Kubernetes clusters need access to secrets (e.g. API keys, database credentials) for their operation, and it's important to ensure that these secrets are managed securely, as unauthorized access to secrets can have serious consequences for organization's security.

## Description

Containerized applications need to have secure access to secret data and there are a number of ways in which this can be inappropriately handled.

- **Secrets in container images.** If a secret is embedded into a container image any attacker who can gain access to the image, will gain access to the credential. It also presents significant difficulties in effectively rotating credentials  
- **Secrets in ConfigMaps.** Whilst ConfigMap objects are handled similarly to Kubernetes Secrets in a number of ways they should not be used to store sensitive data as access control for ConfigMaps is often less strict, and it is more likely that ConfigMaps will be included in debugging information and backups which could lead to unauthorized disclosure.  
- **Secrets in environment variables.** When Kubernetes secrets are used, they can either be presented to the container as environment variables or as a mounted file. In general users should avoid using environment variables as they can be included in debugging information leading to their unauthorized disclosure.  
- **Secrets in code repositories.** Secret information should never be introduced into source code repositories as they may be cloned or replicated to insecure locations resulting in their disclosure.  
- **WATCH/LIST access to secrets.** An important point to know is that any user in the cluster who has LIST or WATCH access to secret objects can effectively retrieve the contents of those secrets.  
- **Secrets in application logs**. Applications often log configuration or environment state during startup or errors. If secrets are handled improperly in code or passed as env variables, they may be written to stdout/stderr. This exposes them to any user with `get` permission on the `pods/log` sub-resource and also to anyone with access to downstream centralized logging platforms (Splunk, Elasticsearch) where they are stored in plaintext, often with long retention periods and weak access controls.

## How to prevent

Secrets should be carefully managed in Kubernetes. There are some controls within Kubernetes which should be used to ensure that they are restricted as much as possible.

- Use either Kubernetes secret objects or an external dedicated secret store. It's important to use purpose built facilities for secret management as it makes the intent clear and users will realise those resources should be secured.  
- If using Kubernetes secrets, consider applying the `--encryption-provider-config` parameter to the API server to encrypt data held in etcd. It's worth noting that this may not provide significant benefit if the API server and etcd process run on the same host(s).  
- Strict RBAC controls should be in place on all access to Secret resources in Kubernetes.  
- Wherever possible eliminate static credentials by adopting Workload Identity Federation or making use of solutions like the External Secrets Operator to inject secrets from a managed vault. This approach allows Pods to authenticate directly to cloud services using their native Kubernetes Service Account via OIDC tokens. By removing the need to store long lived passwords in Kubernetes Secret objects, effectively mitigate the risk of credential theft from etcd or log leaks, while benefiting from short lived, auto rotating tokens that follow the principle of Zero Trust.  
- Long-lived credentials are a primary liability. Use tools like External Secrets Operator or cert-manager to automate rotation. For ServiceAccounts, use the TokenRequest API for short-lived, auto-rotating tokens.

## References

- [Encrypt Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)  
- [Kubernetes Secrets Best Practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)  
- [External Secrets Operator](https://external-secrets.io/)
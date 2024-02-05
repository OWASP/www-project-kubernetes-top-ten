---

layout: col-sidebar
title: "K06: Broken Authentication"
---

## Overview

Authentication in Kubernetes takes on my many forms and is extremely flexible.
This emphasis on being highly configurable makes Kubernetes work in a number of
different environments but also presents challenges when it comes to cluster and
cloud security posture.

![Broken Authentication - Illustration](../../../assets/images/K06-2022.gif)

## Description

Several entities need to access the Kubernetes API. Authentication is the first
hurdle for these requests. Authentication to the Kubernetes API is via HTTP
request and the authentication method can vary from cluster to cluster. If a
request cannot be authenticated, it is rejected with an HTTP status of 401.

![Kubernetes Authentication](../../../assets/images/kubernetes-auth.png)

Source:
[https://kubernetes.io/docs/concepts/security/controlling-access/](https://kubernetes.io/docs/concepts/security/controlling-access/)

Let’s dive into the different types of subjects who need to authenticate to the
Kubernetes API.

### Human Authentication

People need to interact with Kubernetes for a number of reasons. Developers
debugging their running application in a staging cluster, platform engineers
building and testing new infra, and more. There are several methods available to
authenticate to a cluster as a human such as OpenID Connect (OIDC),
Certificates, cloud IAM, and even ServiceAccount tokens. Some of these offer
much more robust security than others as we will explore in the prevention
section below.

### Service Account Authentication

Service account (SA) tokens can be presented to the Kubernetes API as an
authentication mechanism when configured with RBAC appropriately. A SA is a
simple authentication mechanism typically reserved for container-to-api
authentication from *inside* the cluster.

## How to Prevent

### Avoid using certificates for end user authentication

Certificates are convenient to use for authenticating to the Kubernetes API but
should be used with extreme caution. At this time, the API has no way to revoke
certificates making for a scramble to re-key the cluster in the event of a
compromise or leak of private key material. Certificates are also more
cumbersome to configure, sign, and distribute. A certificate may be used as a
“Break Glass” authentication mechanism but not for primary auth.

### Avoid Ingress Object Creation without TLS Certificates

TLS certificates serve as a means of authenticating the server to the
client. When an Ingress Object lacks TLS certificates, there is no
assurance that the server the client is connecting to is legitimate.
This opens the door for man-in-the-middle (MITM) attacks where attackers
can intercept and impersonate the server, potentially leading to data
theft or unauthorized access.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
    - hosts:
        - example.com  # Replace with your domain
      secretName: tls-secret  # Replace with the name of your TLS secret
  rules:
    - host: example.com  # Replace with your domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: your-service-name  # Replace with your backend service name
                port:
                  number: 80  # Replace with your service port number
```

### Never roll your own authentication

Just like crypto, you should not build something novel when it isn’t necessary.
Use what is supported and widely adopted.

### Enforce MFA when possible

No matter the auth mechanism chosen, force humans to provide a second method of
authentication (typically part of OIDC).

### Don’t use Service Account tokens from outside of the cluster

For use inside the cluster, Kubernetes Service Account tokens are
obtained directly using the TokenRequest API, and are mounted into Pods
using a projected volume. For use outside the cluster, these tokens must be
manually provisioned via a [Kubernetes Secret](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-long-lived-api-token-for-a-serviceaccount)
and have no expiration. Using long-lived SA tokens from outside of the cluster
opens your cluster up to significant risk.

If a token based approach is required, short-lived tokens can be provisioned
by the [TokenRequest API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
or using [`kubectl create token` with the `--duration`
flag](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).

### Authenticate users and external services using short-lived tokens

All authentication tokens should be as short-lived as tolerable. This way if
(and when) a credential is leaked, it is possible that it may not be replayed in
the time necessary to compromise the account.

![Broken Authentication -
Mitigations](../../../assets/images/K06-2022-mitigation.gif)

## Example Attack Scenarios

***Accidental Git Leak:*** A developer accidentally checks their `.kubeconfig`
file from their laptop which holds Kubernetes authentication credentials for
their clusters at work. Someone scanning GitHub finds the credentials and
replays them to the target API (unfortunately, sitting on the internet) and
because the cluster is configured to authenticate using certificates, the leaked
file has all of the information needed to successfully authenticate to the
target cluster.

## References

Tremlo Blog Post:
[https://www.tremolosecurity.com/post/what-the-nsa-and-cisa-left-out-of-their-kubernetes-hardening-guide](https://www.tremolosecurity.com/post/what-the-nsa-and-cisa-left-out-of-their-kubernetes-hardening-guide)

Kubernetes Authentication:
[https://kubernetes.io/docs/concepts/security/controlling-access/](https://kubernetes.io/docs/concepts/security/controlling-access/)

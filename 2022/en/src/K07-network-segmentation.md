## Overview
When operating Kubernetes with multiple microservices and tenants, a key area of concern is around control of network traffic. Isolating traffic within the context of a Kubernetes cluster can happen on a few levels including between pods, namespaces, labels, and more. 

![Network Segmentation - Illustration](/assets/images/K07-2022.gif)
 
## Description

Kubernetes networking is flat by default. Meaning that, when no additional controls are in place any workload can communicate to another without constraint. Attackers who exploit a running workload can leverage this default behavior to probe the internal network, traverse to other running containers, or invoke private APIs.

## How to Prevent
Workloads in a cluster eventually need to communicate to one another as well as a variety of internal and external endpoints. The goal of network segmentation within Kubernetes is to minimize the blast radius if a container is compromised and stop lateral movement while allowing valid traffic to route as expected. 

***Native Controls (Multi-Cluster)***: One way to truly enforce network isolation within Kubernetes is to utilize separate clusters when appropriate. This adds complexity when working with tightly coupled microservices but is a viable option when separating different tenants based on risk. 

***Native Controls (NetworkPolicies):*** Network policies are built into Kubernetes itself and behave like firewall rules. They control how pods communicate. Without network policies, any pod can talk to any other pod. Network Policies should be defined as to limit pod communication to only defined assets while denying everything that isn’t explicitly configured. Below is an example of a  network policy prevents backend egress between pods running the “default” namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-backend-egress
  namespace: default

spec:
    podSelector:
    matchLabels:
      tier: backend
      policyTypes:
      - Egress
      egress:
      - to:
         - podSelector:
        matchLabels:
        tier: backend
```

***Service Mesh:*** There are a number of different service mesh projects available for different use cases including [Istio](https://istio.io/), [Linkerd](https://linkerd.io/), and [Hashicorp Consul](https://www.consul.io/docs/k8s). Each of these service mesh technologies offer different ways to segment network traffic within a Kubernetes cluster and all come with pros and cons. Below is an example of an Istio `AuthorizationPolicy`:

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "shoes-writer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: shoes
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/inventory-sa"]
    to:
    - operation:
        methods: ["POST"]
```

- The **`selector`** on **`shoes`** means we're enforcing all Deployments labeled with **`app:shoes`**.
- The **`source`** workload we're allowing has the **`inventory-sa`** identity. In a Kubernetes environment, this means that only pods with the **`inventory-sa`** [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) can access shoes.
- The only allowed HTTP operation is **`POST`**, meaning that other HTTP operations, like **`GET and PUT`** will be denied.

***CNI Plugins:***

Container Network Interface (CNI) is an open source [specification](http://github.com/containernetworking/cni) that is used to configure access to networking resources. CNI is a software-defined mechanism for allowing or disallowing network access within Kubernetes and has a wide variety of supported plugins. Solutions such as [Project Calico](https://www.tigera.io/project-calico/) and [Cilium](https://cilium.io/) all offer different mechanisms for isolating network traffic within the context of Kubernetes. A CNI is typically needed if an operator would like to implement Kubernetes Network Policies (above). 

When choosing a CNI, it is most important to understand the feature-set that you are seeking from a security perspective and the resource overhead and maintenance related to using the plugin. 

![Network Segmentation - Mitigation](/assets/images/K07-2022-mitigation.gif)

## Example Attack Scenarios

A Wordpress pod is compromised on a cluster that has no network segmentation and the attacker is able to utilize built in networking utilities such as `dig` and `curl` to explore the network (it is an Ubuntu base image after all). They discover an internally accessible API running on port `6379` which is typically Redis. They are able to probe the Redis microservice which was intended to be internal and only used by backend APIs using `curl`. Data is stolen and modified. 

A locked-down `NetworkPolicy` or service mesh implementation would have made the network connectivity to Redis from something like Wordpress impossible. 

A not-so-critical web application gets compromised on a cluster which has no network segmentation and the attacker is able to make a request to metadata URL to grab kube-env file containing certificate keys which has all the details for the bootstrap process. The attack can be performed to register itself as a node and steal secrets for further escalation.

A simple `NetworkPolicy` mentioned below can block users from making calls to metadata URL

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-1
spec:
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
  podSelector: {}
  policyTypes:
  - Egress
```

## References

Istio Authorization: [https://istiobyexample.dev/authorization/](https://istiobyexample.dev/authorization/)

Kubernetes CNI Explained: [https://www.tigera.io/learn/guides/kubernetes-networking/kubernetes-cni/](https://www.tigera.io/learn/guides/kubernetes-networking/kubernetes-cni/)

Kubernetes Network Policies: [https://kubernetes.io/docs/concepts/services-networking/network-policies/](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

Hacking kubelet on GKE: [https://www.4armed.com/blog/hacking-kubelet-on-gke/](https://www.4armed.com/blog/hacking-kubelet-on-gke/)

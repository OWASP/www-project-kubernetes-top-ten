### Overview

The security context of a workload in Kubernetes is highly configurable which can lead to serious security misconfigurations propagating across and organization’s workloads and clusters. The [2021 Kubernetes Security Survey](https://www.redhat.com/en/resources/kubernetes-adoption-security-market-trends-2021-overview) from Redhat stated that nearly 60% of respondents have experienced a misconfiguration incident in their Kubernetes environments in the last 12 months. 

### Description

Kubernetes manifests contain many different configurations that can effect the reliability, security, and scalability of a given workload. These configurations should be audited and remediated continuously. Some examples of high-impact manifest configurations are below:

**Application processes should not run as root:** Running the process inside of a container as the `root` user is a common misconfiguration in many clusters. While `root` may be an absolute requirement for some workloads, it should be avoided when possible. If the container were to be compromised, the attacker would have root-level privileges that allow actions such as starting a malicious process that otherwise wouldn’t be permitted with other users on the system. 

```jsx
apiVersion: v1  
kind: Pod  
metadata:  
  name: root-user
spec:  
  containers:  
	...
  securityContext:  
    #root user
    runAsUser: 0
	#non-root user
	runAsUser: 5554	
```


**Read-only filesystems should be used:** In order to limit the impact of a compromised container on a Kubernetes node, it is recommended to utilize read-only filesystems when possible. This prevents a malicious process or application from writing back to the host system. Read-only filesystems are a key component to preventing container breakout.

```jsx
apiVersion: v1  
kind: Pod  
metadata:  
  name: read-only-fs
spec:  
  containers:  

  securityContext:  
	#read-only fs explicitally defined
    readOnlyRootFilesystem: true
```


**Privileged containers should be disallowed**: Setting a container to `privileged` within Kubernetes, the container itself can access additional resources and kernel capabilities of the host itself. Privileged containers are dangerous as they remove many of the built-in container isolation mechanisms entirely. 

```jsx
apiVersion: v1  
kind: Pod  
metadata:  
  name: priviliged-pod
spec:  
  containers:  
	...
  securityContext:  
    #priviliged 
    privileged: true
	#non-priviliged 
	priviliged: false
```

### How to Prevent

Maintaining secure configurations throughout a large, distributed Kubernetes environment can be a difficult task. While many security configurations are often set in the `securityContext` of the manifest itself there are a number of other misconfigurations that can be detected elsewhere. In order to prevent misconfigurations, they must first be detected in both runtime and in code. 

Tools such as Open Policy Agent can be used as a policy engine to detect these common misconfigurations. The CIS Benchmark for Kubernetes can also be used as a starting point for discovering misconfigurations. 


### Example Attack Scenarios



### References

CIS Benchmarks for Kubernetes: [https://www.cisecurity.org/benchmark/kubernetes](https://www.cisecurity.org/benchmark/kubernetes)

Open Policy Agent: [https://github.com/open-policy-agent/opa](https://github.com/open-policy-agent/opa)

Pod Security Standards: [https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

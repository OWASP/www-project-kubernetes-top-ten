### Overview

The security context of a workload in Kubernetes is highly configurable which can lead to serious security misconfigurations propagating across and organization’s workloads and clusters. The [2021 Kubernetes Security Survey](https://www.redhat.com/en/resources/kubernetes-adoption-security-market-trends-2021-overview) from Redhat stated that nearly 60% of respondents have experienced a misconfiguration incident in their Kubernetes environments in the last 12 months. 

### Description

Kubernetes manifests contain many different configurations that can effect the reliability, security, and scalability of a given workload. These configurations should be audited and remediated continuously. Some examples of high-impact manifest configurations are below:

**Application processes should not run as root:** Running the process inside of a container as the `root` user is a common misconfiguration in many clusters. While `root` may be an absolute requirement for some workloads, it should be avoided when possible. If the container were to be compromised, the attacker would have root-level privileges that allow actions such as starting a malicious process that otherwise wouldn’t be permitted with other users on the system. 

```
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

```
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

```
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

*Detect*

Tools such as Open Policy Agent can be used as a policy engine to find these common misconfigurations. The CIS Benchmark for Kubernetes can also be used as a starting point for discovering misconfigurations. 

*Prevent*

Detecting misconfigured workloads is not enough. Teams need the assurance that misconfigured Kubernetes objects can be blocked upon admission. This is typically handled by an [Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) on the Kubernetes API itself. Built-in functionality exists as part of the Kubernetes API itself called Pod Security Standards to enforce policy as part of the [Pod Security Admission Controller](https://kubernetes.io/docs/concepts/security/pod-security-admission/) in the cluster itself. It offers three modes - Privileged, Baseline, and Restricted. Other OSS projects such as Open Policy Agent Gatekeeper, Kyverno, and Kubewarden all offer policy enforcement capabilities as well to prevent misconfigured pods from being scheduled on a cluster. 

### Example Attack Scenarios

Example #1: Container Breakout 1-Liner

The following command if run against the Kubernetes API will create a very special pod that is running a highly privileged container. First we see `"hostPID": true`, which breaks down the most fundamental isolation of containers, letting us see all processes as if we were on the host. The `nsenter` command switches to a different `mount` namespace where `pid 1` is running which is the host `mount` namespace. Finally, we ensure the workload is `priviliged` allowing us to prevent permissions errors. Boom. Container breakout in a [tweet](https://twitter.com/mauilion/status/1129468485480751104https://twitter.com/mauilion/status/1129468485480751104)! 

```
 kubectl run r00t --restart=Never -ti --rm --image lol \
	 --overrides '{"spec":{"hostPID": true, 
	 "containers":[{"name":"1","image":"alpine", 
	 "command":["nsenter","--mount=/proc/1/ns/mnt","--","/bin/bash"], 
     "stdin": true,"tty":true,"imagePullPolicy":"IfNotPresent", 
     "securityContext":{"privileged":true}}]}}' \
/
```

### References

CIS Benchmarks for Kubernetes: [https://www.cisecurity.org/benchmark/kubernetes](https://www.cisecurity.org/benchmark/kubernetes)

Open Policy Agent: [https://github.com/open-policy-agent/opa](https://github.com/open-policy-agent/opa)

Pod Security Standards: [https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

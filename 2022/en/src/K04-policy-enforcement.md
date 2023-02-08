## Overview

Distributing and enforcing security policies across multiple clusters, clouds, and risk tolerances quickly becomes unmanageable for security teams. The inability to detect, remediate, and prevent misconfigurations from a central location can leave clusters open to compromise. 

![Policy Enforcement - Illustration](/assets/images/K04-2022.gif)

## Description
Kubernetes policy enforcement can and should take place in a few places throughout the software delivery lifecycle. Policy enforcement gives security and compliance teams the ability to apply governance, compliance, and security requirements throughout a multi-cluster / multi-cloud infrastructure. 


Example Enforcement Policies:

*Disallowing Images from Untrusted Registries:* To prevent rogue images from running in certain clusters, it is recommended to distribute a blocking admission control policy that explicitly allows image registries. An example OPA Gatekeeper Rego policy that would block all workloads using images from registries that don’t match open-policy-agent and ubuntu is below:

```
# Allowed repos
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "sbx"
      - "prd"
  parameters:
    repos:
      - "open-policy-agent"
      - "ubuntu"
```

## How to Prevent

Detecting misconfigured workloads is not enough. Teams need the assurance that misconfigured Kubernetes objects can be blocked upon admission. This is typically handled by an [Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) on the Kubernetes API itself. Built-in functionality exists as part of the Kubernetes API itself called Pod Security Standards to enforce policy as part of the [Pod Security Admission Controller](https://kubernetes.io/docs/concepts/security/pod-security-admission/) in the cluster itself. It offers three modes - Privileged, Baseline, and Restricted. 

Other OSS projects such as Open Policy Agent Gatekeeper, Kyverno, and Kubewarden all offer policy enforcement capabilities as well to prevent misconfigured pods from being scheduled on a cluster. 

![Policy Enforcement - Mitigations](/assets/images/K04-2022-mitigation.gif)

## Example Attack Scenarios
Example #1: Container Breakout 1-Liner

The following command if run against the Kubernetes API will create a very special pod that is running a highly privileged container. First we see `"hostPID": true`, which breaks down the most fundamental isolation of containers, letting us see all processes as if we were on the host. The `nsenter` command switches to a different `mount` namespace where `pid 1` is running which is the host `mount` namespace. Finally, we ensure the workload is `privileged` allowing us to prevent permissions errors. Boom. Container breakout in a [tweet](https://twitter.com/mauilion/status/1129468485480751104)! 

```
 kubectl run r00t --restart=Never -ti --rm --image lol \
	 --overrides '{"spec":{"hostPID": true, 
	 "containers":[{"name":"1","image":"alpine", 
	 "command":["nsenter","--mount=/proc/1/ns/mnt","--","/bin/bash"], 
     "stdin": true,"tty":true,"imagePullPolicy":"IfNotPresent", 
     "securityContext":{"privileged":true}}]}}' \
/
```

## References
OPA Gatekeeper: [https://github.com/open-policy-agent/gatekeeper](https://github.com/open-policy-agent/gatekeeper)

Pod Security Admission Controller: [https://kubernetes.io/docs/concepts/security/pod-security-admission/](https://kubernetes.io/docs/concepts/security/pod-security-admission/)

Kyverno: [https://kyverno.io/](https://kyverno.io/)

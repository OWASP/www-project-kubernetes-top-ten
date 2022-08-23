### 概述

Kubernetes中工作负载的安全上下文是高度可配置的，这可能会导致严重的安全配置错误在组织的工作负载和集群中传播。在红帽（Redhat）公司的[2021年度Kubernetes安全调查](https://www.redhat.com/en/resources/kubernetes-adoption-security-market-trends-2021-overview) 中表明，近60%的受访者在过去12个月中经历过Kubernetes环境的错误配置事件。

### 问题描述

Kubernetes的描述文件（manifests）中包含许多可以影响工作负载的可靠性、安全性和可伸缩性的配置，应对这些配置持续进行审查和修正。以下是一些包含严重影响的配置示例：

**避免应用程序以根用户身份运行：** 在容器内以`根用户`（root）运行进程是许多集群中常见的错误配置。虽然根用户可能是某些工作负载的刚需，但应尽可能避免使用。如果容器受到危害，攻击者将拥有最高权限（root-level），允许启动恶意进程等操作，而系统上的其他用户则不允许执行这些操作。 

```yaml
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


**使用只读文件系统：** 为了限制有害容器对Kubernetes节点的影响，建议尽可能使用只读文件系统，这也可以防止恶意进程或应用程序对主机的回写（writing back）操作。只读文件系统是防止容器逃脱（breakout）的关键组件。

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: read-only-fs
spec:  
  containers:  
  securityContext:  
  #read-only fs explicitly defined
    readOnlyRootFilesystem: true
```

>  "breakout"是指容器内的应用能够逃避容器的隔离机制，从而访问主机上的资源，如文件系统等 —— 译者注


**禁用特权容器：** 如果在Kubernetes中将容器设置为`特权模式`（privileged），则容器自身可以访问主机的额外资源和内核功能。特权容器非常危险，因为它们完全移除了很多内置的容器隔离机制。

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: privileged-pod
spec:  
  containers:  
	...
  securityContext:  
    #priviliged 
    privileged: true 
    #non-privileged 
    privileged: false
```

### 策略防范

在大型分布式Kubernetes环境中维护安全配置可能是一项艰巨的任务。虽然安全配置通常包含在描述文件自身的`安全上下文`（securityContext） 中，但在其他地方也可能会检测出许多错误配置。为了防止错误配置，它们必须在运行时和代码中首先被检测到。

*开放策略代理*（Open Policy Agent） 等工具可以用作常见错误配置检测的策略引擎。Kubernetes的 *CIS基准测试* 也可以用于发现错误配置。

### 典型攻击场景
略

### 参考资料

Kubernetes CIS基准测试（CIS Benchmarks for Kubernetes）: [https://www.cisecurity.org/benchmark/kubernetes](https://www.cisecurity.org/benchmark/kubernetes)

开放策略代理（Open Policy Agent）: [https://github.com/open-policy-agent/opa](https://github.com/open-policy-agent/opa)

Pod安全标准（Pod Security Standards）: [https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

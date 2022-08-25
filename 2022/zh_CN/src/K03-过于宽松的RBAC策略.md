## 概述
[基于角色的访问控制](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)（Role-Based Access Control，RBAC）是Kubernetes负责资源权限控制的主要机制。这些权限将动词（GET、CREATE、DELETE等）和资源（Pod、服务、节点等）结合在一起，作用域可以是命名空间或集群。Kubernetes 提供了一组开箱即用的角色，这些角色可以根据客户端想要执行的操作默认提供合理的职责分离。强制使用最小权限的RBAC配置是一项挑战，原因将在下面进行探讨。

## 问题描述
如果配置得当，Kubernetes中的RBAC是一种非常强大的安全执行机制。但一旦发生危害，可能会迅速成为群集的巨大风险，并且会造成影响面的扩散。以下是错误配置RBAC的几个案例：

*`cluster-admin`使用不当*

当某个主体（如服务帐户、用户或组）可以访问被称为`集群管理员`（cluster-admin）的Kubernetes内置“超级用户”时，它们能够对集群的全部资源执行任意操作。如果此类拥有集群资源完全控制功能的角色在`集群角色绑定`（ClusterRoleBinding）中被授予了某个对象，则尤其危险。将集群管理员用作`角色绑定`（RoleBinding）也可能会带来重大风险。


如下是流行的OSS Kubernetes开发平台中的RBAC配置。它展示了一个将`cluster-admin`绑定到默认服务帐户的非常危险的`ClusterRoleBinding`。它之所以危险，是因为它将全能的集群管理权限授予给默认命名空间中的每个Pod。如果默认命名空间中的Pod被破坏（例如远程代码执行），那么攻击者可以轻易通过伪造服务来破坏整个集群。 

```yaml
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

## 防范策略

为了降低攻击者滥用RBAC配置的风险，持续分析配置并确保始终贯彻 最小特权原则非常重要。以下是一些建议：

- 尽可能减少终端用户对群集的直接访问
- 避免在集群外部使用服务账户令牌（Service Account Tokens）
- 避免自动加载默认的服务账户令牌
- 为已安装的第三方组件添加RBAC审核功能
- 通过部署集中式策略来检测和阻止有风险的RBAC权限
- 利用`角色绑定`（RoleBindings）将RBAC策略的权限范围限制在特定的命名空间内而非整个群集
- Kubernetes官方文档中的[RBAC最佳实践](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) 

## 典型攻击场景

某OSS集群的可观察性工具被平台工程团队安装在私有的Kubernetes集群内。此工具包含用于调试和流量分析的Web UI 界面。该UI界面碰巧被其服务描述文件中包含的“负载均衡器”（type: LoadBalancer）意外的暴露在互联网上，该类型使用了AWS ALB负载均衡器提供的**公网**IP地址。
假设此工具使用了如下的RBAC配置：

```yaml
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

攻击者找到了开放的Web UI界面，并能够获取群集中正在运行容器的shell。`prd`命名空间中的默认服务帐户令牌由Web UI界面使用，攻击者能够仿冒它来调用Kubernetes API并执行权限提升类操作，例如在`kube-system`命名空间中执行`describe secrets`。这是由于 `roleRef` 为该服务帐户提供了整个集群内置的admin权限。

## 参考资料

Kubernetes RBAC: [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

RBAC Police Scanner: [https://github.com/PaloAltoNetworks/rbac-police](https://github.com/PaloAltoNetworks/rbac-police)

Kubernetes RBAC Good Practices: [https://kubernetes.io/docs/concepts/security/rbac-good-practices/](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
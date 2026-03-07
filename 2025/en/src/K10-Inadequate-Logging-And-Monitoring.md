---

layout: col-sidebar
title: "K10: Inadequate Logging And Monitoring"
---

## Overview

Without appropriate logging and monitoring in place, organizations will not be able to effectively detect attackers or understand the security posture of their clusters. In Kubernetes this is especially important as some sensitive operations leave no permanent record in the cluster datastore, so must be tracked from audit logs.

## Description

Without effective logging, actions taken by attackers in the cluster will be missed and it will be difficult to understand the impact of breaches and incidents.

As Kubernetes clusters are dynamic environments it is particularly challenging to track actions within the cluster, without an effective logging, monitoring, and alerting strategy.

Additionally some key Kubernetes security events, such as the creation of client certificates and service account tokens, are not persisted to the cluster, so audit logging of these events is vital to track potential attackers.

## How to prevent

There are a number of potential logging sources that should be tracked in Kubernetes clusters to understand the security of the workloads running on the cluster and the cluster itself.

At a cluster level the most  important source is Kubernetes audit logging, which tracks operations at the Kubernetes API level. This is often not configured by default in major Kubernetes distributions so care should be taken to ensure that it is enabled and that logs are retained so they can be reviewed when needed.

As Kubernetes does not have a permanent record of credentials that are created for either client certificate or service token authentication, it is vital that creation of these resources is captured in audit logs, so that operators know who created cluster credentials and which credentials exist within the cluster, so that access to the cluster can be effectively tracked. Some information on other items to include in the audit policy are included in the CIS benchmark section 3.2.2.

In addition to auditing the Kubernetes API, logs should be captured from cluster nodes and workloads running on those nodes. When capturing these logs ensure that the tooling understands the dynamic nature of container environments where multiple instances of an application can be spread across many cluster nodes and may change from node to node over time.  
It is also important to consider preventing log loss with implementing a logging architecture that utilizes disk persistent buffering at the node level. This should be with an external message broker to decouple log ingestion from the storage to ensure a reliable and tamper proof audit trail even if downstream services are temporarily unavailable.

## References

- [Kubernetes Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)  
- [Kubernetes Logging](https://kubernetes.io/docs/concepts/cluster-administration/logging/)  
- [CIS Benchmark for Kubernetes](https://www.cisecurity.org/benchmark/kubernetes) \- Sections 3.2.1 and 3.2.2


## Overview

A Kubernetes environment has the ability to generate logs at a variety of levels from many different components. When logs are not captured, stored, or actively monitored attackers have the ability to exploit vulnerabilities while going largely undetected. The lack of logging and monitoring also presents challenges during incident investigation and response efforts. 

![Inadequate Logging - Illustration](/assets/images/K05-2022.gif)

## Description

Inadequate logging in the context of Kubernetes occurs any time:

- Relevant events such as failed authentication attempts, access to sensitive resources, manual deletion or modification of Kubernetes resources are not logged.
- Logs and traces of running workloads are not monitored for suspicious activity.
- Alerting thresholds are not in place or escalated appropriately.
- Logs are not centrally stored and protected against tampering.
- Logging infrastructure is disabled completely.

## How to Prevent

The following logging sources should be enabled and configured appropriately:

**Kubernetes Audit Logs: [Audit logging](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)** is a Kubernetes feature that records actions taken by the API for later analysis. Audit logs help answer questions pertaining to events occurring on the API server itself.

Ensure logs are monitoring for anomalous or unwanted API calls, especially any authorization failures (these log entries will have a status message “Forbidden”). Authorization failures could mean that an attacker is trying to abuse stolen credentials.

Managed Kubernetes providers, including AWS, Azure, and GCP provide optional access to this data in their cloud console and may allow you to set up alerts on authorization failures.

**Kubernetes Events:** Kubernetes events can indicate any Kubernetes resource state changes and errors, such as exceeded resource quota or pending pods, as well as any informational messages.

**Application & Container Logs:** Applications running inside of Kubernetes generate useful logs from a security perspective. The easiest method for capturing these logs is to ensure the output is written to standard output `stdout` and standard error `stderr` streams. Persisting these logs can be carried out in a number of ways. It is common for operators to configure applications to write logs to a log file which is then consumed by a sidecar container to be shipped and processed centrally. 

**Operating System Logs**: Depending on the OS running the Kubernetes nodes, additional logs may be available for processing. Logs from programs such as `systemd` are available using the `journalctl -u` command.

**Cloud Provider Logs:** If you are operating Kubernetes in a managed environment such as AWS EKS, Azure AKS, or GCP GKE you can find a number of additional logging streams available for consumption. One example, is within [Amazon EKS](https://aws.amazon.com/eks/) there exists a log stream specifically for the [Authenticator](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html) component. These logs represent the control plane component that EKS uses for RBAC authentication using AWS IAM credentials and can be a rich source of data for security operations teams. 

**Network Logs:** Network logs can be captured within Kubernetes at a number of layers. If you are working with traditional proxy or ingress components such as nginx or apache, you should use the standard out `stdout` and standard error `stderr` pattern to capture and ship these logs for further investigation. Other projects such as [eBPF](https://ebpf.io/) aim to provide consumable network and kernel logs to greater enhance security observability within the cluster. 

As outlined above, there is no shortage of logging mechanisms available within the Kubernetes ecosystem. A robust security logging architecture should not only capture relevant security events, but also be centralized in a way that is queryable, long term, and maintains integrity.

![Inadequate Logging - Mitigations](/assets/images/K05-2022-mitigation.gif)

## Example Attack Scenarios

Scenario #1: Rouge Insider (anomalous number of “delete” events)

Scenario #2: Service Account Token Compromise

## References

[https://developer.squareup.com/blog/threat-hunting-with-kubernetes-audit-logs/](https://developer.squareup.com/blog/threat-hunting-with-kubernetes-audit-logs/)

[https://kubernetes.io/docs/concepts/cluster-administration/logging/](https://kubernetes.io/docs/concepts/cluster-administration/logging/)

[https://www.cncf.io/blog/2021/12/21/extracting-value-from-the-kubernetes-events-feed/](https://www.cncf.io/blog/2021/12/21/extracting-value-from-the-kubernetes-events-feed/)
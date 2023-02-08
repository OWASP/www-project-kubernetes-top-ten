## Overview
Vulnerabilities exist in Kubernetes and it is up to administrators to follow CVE databases, disclosures, and updates closely and have a plan for patch management. 

![Vulnerable Components - Illustration](/assets/images/K10-2022.gif)
 
## Description

A Kubernetes cluster is an extremely complex software ecosystem that can present challenges when it comes to traditional patch and vulnerability management. 

***ArgoCD CVEs***: ArgoCD is a an extremely popular declarative GitOps tool used to continuously deliver software into one or more clusters. ArgoCD has had a few CVEs over the years including [CVE-2022-24348](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-24348) which allows malicious actors to load a malicious Kubernetes Helm Chart (YAML). ArgoCD runs inside of the cluster and is responsible for deploying these charts automatically. This Helm chart exploits a parsing vulnerability to access restricted information such as API keys, secrets, and more. This data can then be used by the attacker to pivot within the Kubernetes cluster or dump further sensitive data.

***Kubernetes CVEs:***  In October 2021, the popular Kubernetes ingress `ingress-nginx` had a CVE released (https://github.com/kubernetes/ingress-nginx/issues/7837) which allowed users who had the ability to create or update ingress objects the ability to obtain all secrets in a cluster. This leveraged a supported feature called “custom snippets”. This issue was not addressable by solely upgrading the `ingress-nginx` version which made it a scramble for security teams to address at scale. 

***Istio CVEs:*** One of the core pieces of functionality of Istio is to provided service-to-service authN / authZ. In 2020 an authentication bypass vulnerability was discovered ([https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-8595](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-8595)) which abused Istio’s Authentication Policy exact path matching logic. This allowed unauthorized access to resources without a valid JWT token. Attackers could bypass the JWT validation all-together by appending `?` or `#` characters after the protected paths. 

*Minimum Kubernetes Version*: If multiple clusters and running in different cloud or on-prem environments, it is important to maintain an accurate inventory of those clusters and ensure conformance with minimum Kubernetes versions. Using OSS IaC platforms such as Terraform, it is possible to audit the versions of the Kubernetes APIs across multiple clusters and patch when appropriate. 

## How to Prevent

Due to the sheer amount of third-party software running inside of a Kubernetes cluster, it takes a multi-pronged approach to eliminate vulnerable components:

**Track CVE databases:** First and foremost, Kubernetes and the associated components cannot be left out of your existing CVE vulnerability scanning process

**Continuous scanning:** Tools such as OPA Gatekeeper can be used to write custom rules which discover vulnerable  components in a cluster. These should be run on a regular cadence and tracked by the security operations team. 

**Minimize third-party dependencies:** All third-party software should be audited independently before deployment for overly permissive RBAC, low-level kernel access, and historical vulnerability disclosure records. 

![Vulnerable Components - Mitigations](/assets/images/K10-2022-mitigation.gif)

## Example Attack Scenarios

CVEs can be abused in a myriad of ways. If an attacker is able to interact with any of the components within a cluster and exploit the CVE the ramifications can be full cluster compromise. Imagine an attacker taking advantage of a remote code execution vulnerability in a web application and gaining a shell in a cluster. They notice that they are not able to make an HTTP call to an adjacent microservice due to it being protected by an Istio policy. The attacker appends the `?` character to the end of the request which bypasses the JWT validation entirely and is able to pivot to the next protected API. 

## References

ArgoCD CVE Database: [https://www.cvedetails.com/vulnerability-list/vendor_id-11448/product_id-81059/version_id-634305/Linuxfoundation-Argo-cd--.html](https://www.cvedetails.com/vulnerability-list/vendor_id-11448/product_id-81059/version_id-634305/Linuxfoundation-Argo-cd--.html)

CVE Database Keyword “Kubernetes”: [https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=kubernetes](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=kubernetes)

Istio Security Bulletins: [https://istio.io/latest/news/security/](https://istio.io/latest/news/security/)

Kubernetes Security and Disclosure Information: [https://kubernetes.io/docs/reference/issues-security/security/](https://kubernetes.io/docs/reference/issues-security/security/)


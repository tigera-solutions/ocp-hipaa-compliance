# HIPAA compliance demo for Openshift

Implement HIPAA compliance rules using Calico controls.

## Objectives

- [Demo setup](./0-setup/readme.md)
- [Security policies](./1-policies/readme.md)
- [Reporting and visibility](./2-reports/readme.md)
- [End-to-end encryption](./3-encryption/readme.md)
- [Alerts](./4-alerts/readme.md)

## HIPAA compliance rules mapping to Calico capabilities

Mapping of HIPAA compliance rules to Calico controls.

| Security Rule # | Requirement(s) | Calico control(s) |
| --------------- | --------------- | --------------- |
| Access Control <br/>164.312(a)(1) | The ability or the means necessary to read, write, modify, or communicate data/information or otherwise use any system resource. <br/><br/>Implement technical policies and procedures for electronic information systems that maintain electronic protected health information to allow access only to those persons or software programs that have been granted access rights as specified in § 164.308(a)(4) [Information Access Management]. | <ul><li>Calico controls ingress and egress between microservices and external databases, cloud services, APIs, and other applications</li><li>Calico applies least privilege access controls to a cluster, denying all network traffic by default and allowing only those connections that have been authorized</li><li>Calico encrypts data in transit for added protection against data tampering</li><li>Calico organizes all HIPAA endpoints in one or more namespace</li><li>Calico configures the namespace for default-deny and allow-lists all ingress and egress traffic</li></ul> |
| Audit Controls <br/>164.312(b) | Implement hardware, software, and/or procedural mechanisms that record and examine activity in information systems that contain or use electronic protected health information. | <ul><li>Calico continuously monitors and logs all workloads for compliance against existing security policies</li><li>Calico alerts on any configuration changes that may impact existing security policies</li><li>Calico scans container images for vulnerabilities and assigns a pass, warn, or fail label to automatically deploy or block the image from deployment</li><li>Calico provides evidence of compliance with these reports:<ul><li>Inventory report</li><li>Network Access report</li><li>Policy Audit report</li></ul></li><li>Calico provides a runtime view of existing vulnerabilities in your Kubernetes deployment</li><li>Calico’s Elasticsearch logs can be archived to storage services like Amazon S3, Syslog, or Splunk for maintaining and consolidating your compliance data long term</li></ul> |
| Integrity <br/>164.312 (c) | Implement policies and procedures to protect electronic protected health information from improper alteration or destruction | <ul><li>Calico encrypts data in transit for added protection against data tampering</li></ul> |
| Entity Authentication <br/>164.312 (d) | Implement procedures to verify that an entity seeking access to electronic protected health information is the one claimed | <ul><li>Calico controls ingress and egress between microservices and external databases, cloud services, APIs, and other applications</li><li>Calico applies least privilege access controls to a cluster, denying all network traffic by default and allowing only those connections that have been authorized</li></ul> |
| Transmission Security <br/>164.312 (e) | Implement technical security measures to guard against unauthorized access to electronic protected health information that is being transmitted over an electronic communications network | Calico enables you to:<ul><li>Stay current with the network diagram for in-scope workloads in Kubernetes environments using Calico’s Dynamic Service and Threat Graph and flow visualizer</li><li>Allow-list ingress access from the public internet only if the endpoint is providing a publicly accessible service</li><li>Allow-list egress access to the public internet from all in-covered components</li><li>Protect against forged source IP addresses with WireGuard (integrated in Calico)</li><li>Detect and address anomalies and threats with Calico instead of antivirus software</li><li>Report and analyze compliance audit findings with Calico</li><li>Automatically quarantine compromised workloads</li><li>Get insights into statistical and behavioral anomalies with Calico flow logs</li><li>Use policy to implement fine-grained access controls for services</li><li>Detect and prevent OWASP Top 10 attacks with workload-centric WAF</li><li>Monitor, detect and block malicious IPs with integrated AlienVault and custom threat feeds</li><li>Detect malware during runtime using file hashes</li></ul> |

## Cleanup

```bash
kubectl delete -f 1-policies/manifests
kubectl delete -f 1-policies/manifests/1.1-policy-tiers.yaml
kubectl delete -f 2-reports/manifests
kubectl delete -f 4-alerts/manifests
kubectl delete globalnetworkpolicy default.default-deny
kubectl label -n hipstershop pod --all tenant-
kubectl label -n hipstershop pod --all hipaa-
```

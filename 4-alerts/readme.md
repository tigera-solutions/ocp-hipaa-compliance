# Alerts

Alerts allow you to be notified when key events happen in your Kubernetes environment. Calico supports alerting on the following datasets:

- Audit Logs
- DNS Logs
- Flow Logs

>Alerts can be configured either through the CLI or through the Calico Cloud UI.

## Deploy alerts

```bash
# alert any access to SSH port in the hipstershop namespace
oc apply -f 4-alerts/manifests/5.1-ssh-to-hipstershop.yaml
# alert on any changes to the globalnetworkset resource
oc apply -f 4-alerts/manifests/5.2-change-to-globalnetset.yaml
```

## Test alerts

```bash
oc -n hipstershop get po -owide | grep -i frontend
# exec into the tester pod and test connection to frontend_pod and frontend services
oc exec multitool --stdin --tty -- /bin/bash
# from pod's shell run commands
nc -zvw 2 <frontend_pod_IP> 22
```

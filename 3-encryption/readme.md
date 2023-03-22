# Protect data in motion

Calico uses Wireguard to provide end-to-end encryption for pod-to-pod communications.

## Configure Wiregurad dependencies

>Currently, Wireguard dependencies need to be configured on Openshift nodes for Wireguard-based encryption to work. However, going forward Openshift operating system is going to include Wireguard dependencies out of the box which will make this step obsolete.

Follow [Calico documentation](https://docs.tigera.io/calico-enterprise/latest/compliance/encrypt-cluster-pod-traffic) to configure Wireguard dependencies on Openshift nodes.

## Enable encryption

```bash
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'

# verify that encryption is enabled
# there should be projectcalico.org/WireguardPublicKey annotation for each node that has encryption enabled
kubectl get nodes -o yaml | grep 'kubernetes.io/hostname\|Wireguard'

# configure encryption statistics with Prometheus
#kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"nodeMetricsPort":9091}}'
```

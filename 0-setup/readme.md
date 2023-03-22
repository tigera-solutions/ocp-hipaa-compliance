# Demo setup

## Tune Calico settings for the demo

```bash
oc patch felixconfiguration.p default -p '{"spec":{"flowLogsFlushInterval":"10s"}}'
oc patch felixconfiguration.p default -p '{"spec":{"flowLogsFileAggregationKindForAllowed":1}}'
oc patch felixconfiguration.p default -p '{"spec":{"dnsLogsFlushInterval":"10s"}}'
oc patch felixconfiguration default --type='merge' -p '{"spec":{"flowLogsCollectTcpStats":true}}'
```

## Deploy demo application

```bash
# create namespace
oc create namespace hipstershop

# configure elevated permissions for the namespace
# so that you can deploy upstream application stack without the need to adjust securityContext configuration
oc adm policy add-scc-to-user privileged -z default -n hipstershop

# deploy Online Boutique store app
oc apply -n hipstershop -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml

# wait for all pods to rech the Running state
watch oc -n hipstershop get po
```

## Deploy tester app

The `network-multitool` pods will be used in two namespaces to test policy configuration.

```bash
# deploy tester pod into the default namespace
oc apply -f demo/multitool.yaml

# deploy tester pod into the hipsteshop namespace
oc -n hipstershop apply -f demo/multitool.yaml

# verify that both pods are running
oc get pods -A | grep multitool
```

## Test connectivity to the `frontend-external` service

```bash
# check service
oc get svc -n hipstershop frontend-external

# connect to the tester pod and test connectivity to the frontend-external service
oc exec multitool --stdin --tty -- /bin/bash
# from inside of the pod's shell use IP of the service
curl -I 10.49.14.192 # <-- replace IP with the service IP in your cluster
```

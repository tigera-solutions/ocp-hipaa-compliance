# Observability and Reporting

With the policy portion of securing the application completed, we need a way to report that the application is in compliance going forward. There are two main tools for this within Calico Cloud: Dynamic Service Graph and Flow Visualizations.

## Observability

Navigate to the Service Graph in the UI and explore the `hipstershop` namespace.

Then, navigate to the Flow Visualizations tool in the UI and explore the connections flowing though the cluster.

Calico Cloud also includes Kibana component to view the raw log data for the traffic within your cluster. Kibana provides its own set of powerful filtering capabilities to quickly drill into log data. For example, use filters to drill into flow log data for specific namespaces and pods. Or view details and metadata for a single flow log entry.

## Reporting

Using the reporting feature of Calico Cloud we can create a number of reports to satisfy the various HIPAA reporting requirements.

Calico Cloud supports the following built-in report types:

- Inventory
- Network Access
- Policy-Audit
- CIS Benchmark

These reports can be customized to report against a certain set of endpoints (for example HIPAA endpoints).

Compliance reports provide the following high-level information:

- Protection
  - Endpoints explicitly protected using ingress or egress policy
  - Endpoints with Envoy enabled
- Policies and services
  - Policies and services associated with endpoints
  - Policy audit logs
- Traffic
  - Allowed ingress/egress traffic to/from namespaces
  - Allowed ingress/egress traffic to/from the internet

### CIS Benchmark Reports

CIS Benchmarks are best practices for the secure configuration of a target system - [Center for Internet Security](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq).

Being able to assess your Kubernetes clusters against CIS benchmarks is a standard requirement for an organization’s security and compliance posture. Calico Enterprise CIS benchmark compliance reports provide this view into your Kubernetes clusters.

### Configure reports

Review and configure example reports:

```bash
# daily CIS Benchmark reporting
oc apply -f 2-reports/manifests/2.1-cis-benchmarks.yaml

# weekly full infrastructure inventory reporting - the report will only include nodes with label `nodetype=infrastructure`.
oc apply -f 2-reports/manifests/2.2-infra-inventory.yaml

# daily report - Tenant Endpoint Network Access
oc apply -f 2-reports/manifests/2.3-tenant-network-access.yaml

# daily report - HIPAA Endpoint Inventory
oc apply -f 2-reports/manifests/2.4-hipaa-inventory.yaml

# daily report - HIPAA Endpoint Network Access
oc apply -f 2-reports/manifests/2.5-hipaa-network-access.yaml

# daily report - HIPAA Policy Audit
oc apply -f 2-reports/manifests/2.6-hipaa-policy-audit.yaml

# hourly report - Full Inventory
oc apply -f 2-reports/manifests/2.7-full-inventory.yaml
```

### Generate reports manually

```bash
# get Calico version
CALICO_VERSION=$(oc get clusterinformation default -ojsonpath='{.spec.cnxVersion}')
# set report names
CIS_REPORT_NAME='daily-cis-results'
INVENTORY_REPORT_NAME='daily-hipaa-inventory'
NETWORK_ACCESS_REPORT_NAME='daily-hipaa-network-access'
# for managed clusters you must set ELASTIC_INDEX_SUFFIX var to cluster name in the reporter pod template YAML
ELASTIC_INDEX_SUFFIX=$(oc get deployment -n tigera-intrusion-detection intrusion-detection-controller -ojson | jq -r '.spec.template.spec.containers[0].env[] | select(.name == "CLUSTER_NAME").value')

# on MacOS
if [[ $(uname -s) == 'Darwin' ]]; then
  START_TIME=$(date -v -2H -u +'%Y-%m-%dT%H:%M:%SZ')
  END_TIME=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
else
# on Linux
  START_TIME=$(date -d '-2 hours' -u +'%Y-%m-%dT%H:%M:%SZ')
  END_TIME=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
fi

# replace variables in YAML and deploy reporter jobs
sed -e "s?<CALICO_VERSION>?$CALICO_VERSION?g" \
  -e "s?<CIS_REPORT_NAME>?$CIS_REPORT_NAME?g" \
  -e "s?<INVENTORY_REPORT_NAME>?$INVENTORY_REPORT_NAME?g" \
  -e "s?<NETWORK_ACCESS_REPORT_NAME>?$NETWORK_ACCESS_REPORT_NAME?g" \
  -e "s?<ELASTIC_INDEX_SUFFIX>?$ELASTIC_INDEX_SUFFIX?g" \
  -e "s?<REPORT_START_TIME_UTC>?$START_TIME?g" \
  -e "s?<REPORT_END_TIME_UTC>?$END_TIME?g" \
  2-reports/manifests/2.8-reporter-pods.yaml | oc apply -f -
```

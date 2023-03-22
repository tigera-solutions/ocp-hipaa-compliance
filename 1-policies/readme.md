# Security policies

>**Goal**: Now that we've deployed our web store application, in order to comply with the PCI DSS standard we have to secure it. This means applying Security Policies to restrict access as a much as possible. In this section we will walk through building up a robust security policy for our application.

## Configure labels

Instead of IP addresses and IP ranges, network policies in Kubernetes depend on labels and selectors to determine which workloads can talk to each other. Workload identity is the same for Kubernetes and Calico Cloud network policies: as pods dynamically come and go, network policy is enforced based on the labels and selectors that you define.

Assign `hipaa=true` label to all pods in the `hipstershop` namespace

```bash
oc label pods --all -n hipstershop hipaa=true

# verify label configuration
oc get pods -n hipstershop --show-labels | grep hipaa
```

## Configure Policy Tiers

Tiers are a hierarchical construct used to group policies and enforce higher precedence policies that cannot be circumvented by other teams. As part of your microsegmentation strategy, tiers let you apply identity-based protection to workloads and hosts. All Calico Enterprise and Kubernetes network policies reside in tiers.

For this workshop, we'll be creating 3 tiers in the cluster and utilizing the default tier as well:

**security** - Global security tier with controls such as PCI restrictions.

**platform** - Platform level controls such as DNS policy and tenant level isolation.

**app-hipstershop** - Application specific tier for microsegmentation inside the application.

To create the tiers apply the following manifest:

```bash
oc apply -f 1-policies/manifests/1.1-policy-tiers.yaml
```

## Configure general policies

After creating our tiers, we'll apply some general policies to them before we start creating our main policies. These policies include allowing traffic to kube-dns from all pods, passing traffic that doesn't explicitly match in the tier and finally a default deny policy.

```bash
oc apply -f 1-policies/manifests/1.2-security-pass.yaml
oc apply -f 1-policies/manifests/1.3-platform-pass.yaml
oc apply -f 1-policies/manifests/1.4-allow-platform-dns.yaml
oc apply -f 1-policies/manifests/1.5-default-deny.yaml
```

## Configure security policies

Now that there are base layer policies in the Policy Tiers, we can configure more restrictive policies to control traffic. The first policy we will apply will only allow traffic to flow between pods with the label of `hipaa=true`. Pods without the `hipaa=true` label will also be able to freely communicate with each other.

We will also add a `hipaa-allowlist` policy because we need a way to allow traffic to the frontend of the application as well as allowing DNS lookups from the HIPAA compliant pods to the `openshift-dns` system.

```bash
oc apply -f 1-policies/manifests/1.6-hipaa-isolation.yaml
```

## Test HIPAA compliant policy

To test, we'll use our `multitool` pods both inside of the `hipstershop` namespace and in the `default` namespace. Before we can complete the testing from the `default` namespace, we'll have to apply a policy that allows egress traffic from the pods in the `default` namespace. This is because we're applying an egress policy in an earlier step, so now, if we don't allow it explicitly, the default deny will drop the traffic when it is enforced. To get around this we'll apply this policy:

```bash
oc apply -f 1-policies/manifests/1.7-default-egress.yaml
```

Get the addresses of all the Online Boutique services so we can use them in the testing to follow. To do this we'll run the following command and keep the output handy:

```bash
oc get svc -n hipstershop -o wide
```

First, from inside of the `hipstershop` namespace, exec into the multitool pod and connect to the `frontend` and `cartservice` services directly. To do this use `netcat` and `curl`.

You can get `cartservice` IP with the following command:

```bash
```

Test connectivity within the `hipstershop` namespace:

```bash
# get cartservice IP
echo "cartservice IP"
oc get svc -n hipstershop cartservice -ojsonpath='{.spec.clusterIP}'
echo ""

# get frontend service IP
echo "frontend service IP"
oc get svc -n hipstershop frontend -ojsonpath='{.spec.clusterIP}'

# exec into the tester pod and test connection to cartservice and frontend services
oc exec -n hipstershop multitool --stdin --tty -- /bin/bash
# from pod's shell run commands
nc -zvw 2 <cartservice_IP> 7070
curl -m2 -I <frontend_IP>

# exit from the pod's shell
```

As expected, we can reach both services from a pod with the `hipaa=true` label.

Next, test connectivity from a pod that doesn't have `hipaa=true` label configured, such as `multitool` pod in the `default` namespace:

```bash
# exec into the tester pod and test connection to cartservice and frontend services
oc exec multitool --stdin --tty -- /bin/bash
# from pod's shell run commands
nc -zvw 2 <cartservice_IP> 7070
curl -I <frontend_IP>

# exit from the pod's shell
```

As expected, we can connect to `frontend` because it has a policy allowing it but we can't connect to the `cartservice` on `7070` because of our HIPAA isolation policy.

Let's add the `hipaa=true` label to the `default/multitool` pod and test the connectivity once again:

```bash
# add label
oc label pod multitool hipaa=true

# exec into the tester pod and test connection to cartservice and frontend services
oc exec multitool --stdin --tty -- /bin/bash
# from pod's shell run commands
nc -zvw 2 <cartservice_IP> 7070
```

This time we can successfully connect from the `multitool` pod in the `default` namespace to a service in the `hipstershop` namespace as long as they both have the `hipaa=true` label.

## Tenant isolation

If we need to further isolate the workloads in the cluster, for example in the case of multi-tenancy, we can start isolating based on properties such as namespace or a tenant label.

Let's restrict access between those `hipaa=true` resources even further starting with isolating the `hipstershop` namespace from outside traffic other than the `frontend` traffic.

To accomplish this, we will isolate our tenants within the cluster. In this case, the `hipstershop` will be a single tenant so we'll attach a tenant label to all of it's pods:

```bash
oc label -n hipstershop pod --all tenant=hipstershop
```

Now, we can create a policy that only allows communication within this tenant label:

```bash
oc apply -f 1-policies/manifests/1.8-tenant-isolation.yaml

# update hipaa-restrict policy to let tenant isolation policy do policy enforcement
oc apply -f 1-policies/manifests/1.9-hipaa-policy-update.yaml
```

There are multiple ways to accomplish this, we could have very easily isolated traffic within the namespace as well. Isolating based on the tenant label in this scenario accomplishes our goal of isolating traffic within the `hipstershop` application. We can verify by again testing using `curl` and `netcat`:

Testing from outside of the tenant label (`multitool` pod in the `default` namespace):

```bash
# exec into the tester pod and test connection to cartservice and frontend services
oc exec multitool --stdin --tty -- /bin/bash
# from pod's shell run commands
nc -zvw 2 <cartservice_IP> 7070
curl -m2 -I <frontend_IP>
```

Connection to the `cartservice` is not allowed by tenant isolation policy, but the connection to the `frontend` service is still allowed by `hipaa=allowlist` rule.

## Microsegmentation with `hipstershop` application

To configure identity-aware microsegmentation, we need to know the relationship between the hipstershop services. The following table describes it:

Source Service | Destination Service | Destination Port
--- | --- | ---
cartservice | redis-cart | 6379
checkoutservice | cartservice | 7070
checkoutservice | emailservice | 8080
checkoutservice | paymentservice | 50051
checkoutservice | productcatalogservice | 3550
checkoutservice | shippingservice | 50051
checkoutservice | currencyservice | 7000
checkoutservice | adservice | 9555
frontend | cartservice | 7070
frontend | productcatalogservice | 3550
frontend | recommendationservice | 8080
frontend | currencyservice | 7000
frontend | checkoutservice | 5050
frontend | shippingservice | 50051
frontend | adservice | 9555
loadgenerator | frontend | 8080
recommendationservice | productcatalogservice | 3550

Now, let's configure the policy that provides microsegmentation within the `app-hipstershop` tier:

```bash
oc apply -f 1-policies/manifests/1.10-hipstershop-microsegmentation.yaml
```

In order to enable microsegmentation decisions to apply at `app-hipstershop` tier, we need to adjust tenant isolation policy to pass policy enforcement to the microsegmentation policies.

```bash
oc apply -f 1-policies/manifests/1.11-tenat-isolation-update.yaml
```

Now, the communications between `hipstershop` services are restricted to only necessary services. You shouldn't be able to connect to the `frontend` service from the `multitool` tester pod in the `hipstershop` namespace.

## Restrict egress access

Now that we've implemented our microsegmentation policy, there's one last type of policy we should apply - a global egress access policy.

A global egress access policy allows us to limit what external resources the pods in our cluster can reach. To build this we need two pieces:

1. A GlobalNetworkSet with a list of approved external domains.
2. An egress policy that applies globally and references our GlobalNetworkSet.

First, create a list of allowed domains:

```bash
oc apply -f 1-policies/manifests/1.12-global-trusted-domains.yaml
```

And now we'll apply our policy into the security tier and have it reference our list of trusted domains we just created.

```bash
oc apply -f 1-policies/manifests/1.13-trusted-domains-policy.yaml
```

Now any pod that doesn't have a more permissive egress policy will only be allowed to access `google.com` and `tigera.io` and we can test this with our `multitool` pod in the `hisptershop` namespace.

Exec into the `multitool` pod in the `hipstershop` namespace:

```bash
# exec into the tester pod and test connection to cartservice and frontend services
oc exec -n hipstershop multitool --stdin --tty -- /bin/bash
# from pod's shell run commands
# connect to google.com
ping -c 2 google.com
# connect to tigera.io
ping -c 2 tigera.io
# connect to github.com
ping -c 2 github.com

# exit from the pod's shell
```

As expected our pings to `google.com` and `tigera.io` are successful but our ping to `github.com` is denied.

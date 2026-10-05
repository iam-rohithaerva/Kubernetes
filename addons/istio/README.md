# How to Install Istio and Secure, Route, and Observe Service Traffic

This guide installs Istio on a Kubernetes cluster and uses it on a small demo app (`frontend`, `orders` v1/v2, `payments`). It covers what Istio is, the problems it solves, how to implement mTLS, zero trust access, and canary routing, how to verify each one, and what gets created in the cluster at every step.

---

## Table of Contents

1. [What Istio Is](#1-what-istio-is)
2. [What Problems It Solves](#2-what-problems-it-solves)
3. [Environment](#3-environment)
4. [Files in This Folder](#4-files-in-this-folder)
5. [Step 1: Install istioctl](#5-step-1-install-istioctl)
6. [Step 2: Install Istio in the Cluster](#6-step-2-install-istio-in-the-cluster)
7. [Step 3: Deploy the Demo App with Sidecars](#7-step-3-deploy-the-demo-app-with-sidecars)
8. [Step 4: Enforce mTLS](#8-step-4-enforce-mtls)
9. [Step 5: Zero Trust with AuthorizationPolicy](#9-step-5-zero-trust-with-authorizationpolicy)
10. [Step 6: Canary Release](#10-step-6-canary-release)
11. [Step 7: Ingress Gateway](#11-step-7-ingress-gateway)
12. [Step 8: Tracing and Observability](#12-step-8-tracing-and-observability)
13. [What Gets Created in the Cluster](#13-what-gets-created-in-the-cluster)
14. [Request Flow: Before and After Istio](#14-request-flow-before-and-after-istio)
15. [Production Checklist](#15-production-checklist)
16. [Troubleshooting](#16-troubleshooting)
17. [Cleanup](#17-cleanup)

---

## 1. What Istio Is

Istio is an open-source **service mesh** that runs as an add-on on top of Kubernetes. A service mesh is an infrastructure layer that handles service-to-service traffic: encryption, retries, routing, and metrics move out of application code and into a proxy that runs next to every pod.

```
┌──────────── Pod: orders ────────────┐        ┌──────────── Pod: payments ──────────┐
│  orders app  ⇄  Envoy sidecar       │ ─mTLS─▶│  Envoy sidecar  ⇄  payments app     │
└─────────────────────────────────────┘        └─────────────────────────────────────┘
                         ▲  routes, policies, certificates  ▲
                         └──────────────  istiod  ──────────┘
```

| Part | Component | Job |
|---|---|---|
| Control plane | `istiod` | Watches Istio CRDs, converts them into Envoy config, issues and rotates workload certificates |
| Data plane | Envoy sidecars (`istio-proxy`) | Carry the actual traffic and enforce the config pushed by istiod |

Applications do not change. They still call `http://payments:8080`; the sidecar intercepts the call and applies mTLS, routing, retries, and telemetry.

## 2. What Problems It Solves

Without a mesh, every team implements retries, TLS, and metrics in every language, and the cluster network is flat and unencrypted.

| Area | Without Istio | With Istio |
|---|---|---|
| **Traffic** | Canary is approximated with replica counts; retries and timeouts live in app libraries | Exact weighted splits (90/10), header-based routing, retries, timeouts, and fault injection in YAML |
| **Security** | Pod-to-pod traffic is plain text; any pod can call any pod | Automatic mTLS with certificate rotation; identity-based AuthorizationPolicy (zero trust); JWT validation |
| **Observability** | Each service logs differently; finding the slow hop means digging through logs | Uniform metrics for every hop (Prometheus/Grafana), distributed traces (Jaeger/Tempo), live service graph (Kiali) |
| **Resilience** | One slow dependency cascades into an outage | Circuit breaking, outlier detection that ejects bad pods, connection pool limits |

Trade-off: every sidecar uses extra CPU and memory and adds a small amount of latency. Istio ambient mode (ztunnel plus waypoint proxies, no sidecars) reduces this overhead.

Kubernetes alone can partly cover security: NetworkPolicy restricts traffic by IP and port, and CNIs such as Cilium or Calico can encrypt node-to-node traffic with WireGuard. Istio adds workload identity, mutual authentication, and L7 rules (HTTP method and path) without application changes.

## 3. Environment

| Item | Value |
|---|---|
| Kubernetes | Any conformant cluster (EKS, kubeadm, kind, minikube). Check the Istio support matrix for your version |
| Istio | Latest stable release, `default` profile |
| Client machine | Laptop, bastion, or CI runner with a working `kubectl` context |
| Demo namespace | `shop` (sidecar injection on), `legacy` (no injection, used for negative tests) |
| Demo host | `shopkart.example.com` |

## 4. Files in This Folder

| File | Contents |
|---|---|
| [manifests/00-namespace.yaml](manifests/00-namespace.yaml) | `shop` namespace with `istio-injection=enabled`, and the `legacy` namespace |
| [manifests/01-workloads.yaml](manifests/01-workloads.yaml) | ServiceAccounts, `frontend` (curl client), `orders` v1 and v2, `payments`, and their Services |
| [manifests/02-peer-authentication.yaml](manifests/02-peer-authentication.yaml) | STRICT mTLS for the `shop` namespace |
| [manifests/03-authorization-policy.yaml](manifests/03-authorization-policy.yaml) | Deny-all plus allow rules based on ServiceAccount identity |
| [manifests/04-orders-canary.yaml](manifests/04-orders-canary.yaml) | DestinationRule subsets and VirtualService with 90/10 weights, beta header route, retries |
| [manifests/05-gateway.yaml](manifests/05-gateway.yaml) | Ingress Gateway for `shopkart.example.com` |
| [manifests/06-telemetry.yaml](manifests/06-telemetry.yaml) | Mesh-wide trace sampling |

The `orders` and `payments` containers are `hashicorp/http-echo`, which reply with fixed text (`orders-v1`, `orders-v2`, `payment-ok`). That makes the canary split and the access checks visible in plain curl output.

---

## 5. Step 1: Install istioctl

`istioctl` is a client CLI, like `kubectl`. Install it on the machine that has your kubeconfig, **not on the cluster nodes**. It talks to the Kubernetes API server; everything Istio runs in the cluster runs as pods.

```bash
# macOS
brew install istioctl

# Linux or macOS, official script (also downloads samples/ with Kiali, Jaeger, Prometheus addons)
curl -L https://istio.io/downloadIstio | sh -
cd istio-*/
export PATH=$PWD/bin:$PATH

# Pin a version if needed
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=<version> sh -
```

Verify:

```bash
kubectl config current-context   # confirm you are pointing at the right cluster
istioctl version                 # client version; "no running Istio pods" is expected before install
istioctl x precheck              # checks that the cluster can run Istio
```

Keep the `istioctl` version equal to the Istio version in the cluster. Upgrades are done with the matching `istioctl`.

## 6. Step 2: Install Istio in the Cluster

```bash
istioctl install --set profile=default -y
```

The `default` profile installs `istiod` and an ingress gateway. Verify:

```bash
kubectl get pods -n istio-system
# istiod-xxxx                   1/1 Running
# istio-ingressgateway-xxxx     1/1 Running

kubectl get svc -n istio-system istio-ingressgateway   # type LoadBalancer; on EKS this creates an AWS load balancer
kubectl get crds | grep istio.io
```

In production, most teams install with the Helm charts (`istio/base`, `istio/istiod`, `istio/gateway`) through GitOps (Argo CD or Flux) and keep `istioctl` for prechecks, analysis, and debugging.

## 7. Step 3: Deploy the Demo App with Sidecars

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/01-workloads.yaml
```

Verify that every pod has two containers (the app and `istio-proxy`):

```bash
kubectl get pods -n shop
# NAME                         READY   STATUS
# frontend-xxxx                2/2     Running
# orders-v1-xxxx               2/2     Running
# orders-v2-xxxx               2/2     Running
# payments-xxxx                2/2     Running
```

If you label an existing namespace, already running pods do not get a sidecar until they restart:

```bash
kubectl label namespace <ns> istio-injection=enabled
kubectl rollout restart deployment -n <ns>
```

## 8. Step 4: Enforce mTLS

By default Istio runs in PERMISSIVE mode: sidecars accept both mTLS and plain text, so existing clients keep working. STRICT rejects plain text.

```bash
kubectl apply -f manifests/02-peer-authentication.yaml
```

Roll out PERMISSIVE first in real clusters, confirm in Kiali or metrics that all traffic is mTLS, then switch to STRICT. A client without a sidecar breaks as soon as STRICT is applied.

**Verify: plain text from outside the mesh is rejected**

```bash
kubectl run t -n legacy --image=curlimages/curl:8.10.1 -it --rm --restart=Never -- \
  curl -sS -m 5 http://orders.shop:8080
# curl: (56) Recv failure: Connection reset by peer   <- expected
```

**Verify: workload certificates are issued**

```bash
istioctl proxy-config secret deploy/orders-v1 -n shop
# default   Cert Chain   ACTIVE   true   <serial>   <expiry>
# ROOTCA    CA           ACTIVE   true   <serial>   <expiry>
```

## 9. Step 5: Zero Trust with AuthorizationPolicy

```bash
kubectl apply -f manifests/03-authorization-policy.yaml
```

| Policy | Effect |
|---|---|
| `deny-all` | Empty spec: every request to the namespace is denied unless another policy allows it |
| `orders-allow` | `orders` accepts calls from the `frontend` ServiceAccount and the ingress gateway |
| `payments-allow` | `payments` accepts only `POST /pay` from the `orders` ServiceAccount |

The `principals` field is the ServiceAccount identity carried in the mTLS certificate (`cluster.local/ns/<namespace>/sa/<serviceaccount>`). This is why every workload needs its own ServiceAccount, and why AuthorizationPolicy with principals requires mTLS.

**Verify: allowed call succeeds**

```bash
kubectl exec -n shop deploy/frontend -c frontend -- curl -s http://orders:8080
# orders-v1  (or orders-v2)
```

**Verify: denied call returns 403**

```bash
kubectl exec -n shop deploy/frontend -c frontend -- curl -s -w '\n%{http_code}\n' -X POST http://payments:8080/pay
# RBAC: access denied
# 403
```

`frontend` is not allowed to call `payments`; only `orders` is.

## 10. Step 6: Canary Release

How the pieces fit:

| Object | Role |
|---|---|
| Two Deployments | `orders-v1` and `orders-v2`, same `app: orders` label, different `version` label |
| One Service | Selects `app: orders` only, so it covers both versions |
| DestinationRule | Names the subsets `v1` and `v2` by the `version` label |
| VirtualService | Sends a weight of traffic to each subset, routes beta users to v2, adds retries and timeout |

Weights are independent of replica counts: one v2 pod still receives exactly 10%.

```bash
kubectl apply -f manifests/04-orders-canary.yaml
```

In a real release, apply the VirtualService with weights `100/0` **before** deploying v2. Otherwise the Service round-robins across all pods and v2 receives traffic as soon as it starts.

**Verify: the split**

```bash
kubectl exec -n shop deploy/frontend -c frontend -- sh -c \
  'for i in $(seq 100); do curl -s http://orders:8080; done' | sort | uniq -c
#   90 orders-v1
#   10 orders-v2   (approximately)
```

**Verify: header routing**

```bash
kubectl exec -n shop deploy/frontend -c frontend -- sh -c \
  'for i in $(seq 10); do curl -s -H "x-user-type: beta" http://orders:8080; done' | sort | uniq -c
#   10 orders-v2
```

**Promote or roll back** by editing the weights in `04-orders-canary.yaml` and applying again. Typical sequence: `100/0`, `90/10`, `50/50`, `0/100`. Rollback is `100/0`; it takes effect in seconds with no pod restart. After a full promotion, delete `orders-v1` and point the route only at v2.

In production, automate the weight changes with Argo Rollouts or Flagger, which promote step by step and roll back automatically when Prometheus metrics (error rate, latency) cross a threshold.

## 11. Step 7: Ingress Gateway

With Istio, a separate NGINX ingress controller is not needed. The `istio-ingressgateway` deployment (Envoy) receives external traffic, `istiod` programs it, and two CRDs configure it:

| NGINX Ingress | Istio |
|---|---|
| NGINX controller pod | `istio-ingressgateway` pod |
| `Ingress` host and TLS | `Gateway` |
| `Ingress` path rules | `VirtualService` bound to the Gateway |

```bash
kubectl apply -f manifests/05-gateway.yaml
```

The `orders` VirtualService already lists `shop-gw` in `gateways` and `shopkart.example.com` in `hosts`, so the same canary and retry rules apply to external traffic.

**Verify**

```bash
GW=$(kubectl get svc -n istio-system istio-ingressgateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{.status.loadBalancer.ingress[0].ip}')
curl -s -H "Host: shopkart.example.com" http://$GW/
# orders-v1  (or orders-v2)
```

On kind or minikube without a load balancer, use `kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80` and curl `localhost:8080`.

Advantages over a plain ingress controller: canary weights, retries, and header routing are available at the edge; traffic stays encrypted with mTLS from the gateway to the pod; edge and internal traffic share the same metrics and traces. If you need AWS WAF or Cognito, put an ALB in front of the ingress gateway.

The Kubernetes Gateway API (`Gateway` and `HTTPRoute` in `gateway.networking.k8s.io`) is the standard alternative to the Istio Gateway and VirtualService. Istio supports it, and ambient mode relies on it. Its CRDs are installed separately.

## 12. Step 8: Tracing and Observability

```bash
kubectl apply -f manifests/06-telemetry.yaml

# Demo addons from the downloaded Istio folder (not for production)
kubectl apply -f istio-*/samples/addons/prometheus.yaml
kubectl apply -f istio-*/samples/addons/jaeger.yaml
kubectl apply -f istio-*/samples/addons/kiali.yaml
```

Depending on the Istio version, Jaeger may also need to be registered as a tracing provider in the mesh config. See the Jaeger task in the Istio docs for your version.

Generate some traffic, then open the dashboards:

```bash
istioctl dashboard kiali     # Graph -> namespace shop: lock icons mean mTLS, red edges mean errors
istioctl dashboard jaeger    # http://localhost:16686
```

Finding a slow request in Jaeger:

1. Service: `istio-ingressgateway.istio-system` (start at the entry point to see the full chain)
2. Min Duration: for example `3s`; or Tags: `http.status_code=500`
3. Find Traces, open one, and look for the longest span in the waterfall. That hop is the bottleneck.
4. For a specific complaint, search by trace ID or `x-request-id`.

Envoy records every hop, but the application must **forward the trace headers** (`traceparent`, `x-request-id`, `x-b3-*`) on its outgoing calls; otherwise each hop appears as a separate trace. The OpenTelemetry SDK does this automatically.

In production, send spans through an OpenTelemetry Collector to Grafana Tempo, AWS X-Ray, or another backend, and sample 1 to 5 percent.

## 13. What Gets Created in the Cluster

| Action | What is created or changed |
|---|---|
| `istioctl install` | `istio-system` namespace; `istiod` Deployment; `istio-ingressgateway` Deployment and LoadBalancer Service; Istio CRDs; mutating webhook for sidecar injection; validating webhook; mesh root CA |
| Label namespace `istio-injection=enabled` | Nothing yet. The webhook now watches pods created in this namespace |
| Create or restart pods | Webhook adds the `istio-proxy` (Envoy) container and an init container (`istio-init`, or the Istio CNI plugin) that redirects pod traffic through Envoy with iptables. Pods show `2/2` |
| Apply PeerAuthentication | istiod pushes mTLS settings to the sidecars |
| Apply AuthorizationPolicy | istiod pushes RBAC filters to the selected sidecars |
| Apply VirtualService / DestinationRule / Gateway | istiod translates them into Envoy routes, clusters, and listeners and pushes them over xDS. No restart |

Istio CRDs by API group:

| API group | CRDs |
|---|---|
| `networking.istio.io` | Gateway, VirtualService, DestinationRule, ServiceEntry, Sidecar, EnvoyFilter, WorkloadEntry, WorkloadGroup, ProxyConfig |
| `security.istio.io` | PeerAuthentication, RequestAuthentication, AuthorizationPolicy |
| `telemetry.istio.io` | Telemetry |
| `extensions.istio.io` | WasmPlugin |

```bash
kubectl get crds | grep istio.io
kubectl api-resources | grep istio
```

Which CRDs to use for what:

| Use case | CRDs |
|---|---|
| mTLS and zero trust | PeerAuthentication, AuthorizationPolicy |
| End-user JWT | RequestAuthentication, AuthorizationPolicy |
| External traffic in | Gateway, VirtualService |
| Canary or blue-green | DestinationRule, VirtualService |
| Retries and timeouts | VirtualService |
| Circuit breaking | DestinationRule (`outlierDetection`, `connectionPool`) |
| Calls to services outside the mesh | ServiceEntry |
| Tracing and logging | Telemetry |
| Reduce sidecar memory | Sidecar |

## 14. Request Flow: Before and After Istio

```
BEFORE                                          AFTER
User                                            User
 │ HTTPS                                         │ HTTPS
 ▼                                               ▼
NGINX Ingress (TLS ends here)                   istio-ingressgateway (Gateway + VirtualService)
 │ plain HTTP                                    │ mTLS
 ▼                                               ▼
frontend                                        [envoy] frontend
 │ plain HTTP                                    │ mTLS, 90/10 split, retries
 ▼                                               ▼
orders                                          [envoy] orders v1 / v2
 │ plain HTTP, any pod may call                  │ mTLS, AuthorizationPolicy checked
 ▼                                               ▼
payments                                        [envoy] payments
```

| | Before | After |
|---|---|---|
| Encryption inside the cluster | None | mTLS on every hop |
| Who can call payments | Any pod | Only the `orders` ServiceAccount, only `POST /pay` |
| Canary | Approximate, by replica count | Exact weights, instant rollback |
| Retries and timeouts | Code in each app | VirtualService |
| Debugging a slow request | Logs from each service | One trace across all hops |

## 15. Production Checklist

- [ ] Install with Helm through GitOps and pin the Istio version
- [ ] Run `istioctl x precheck` and check the Kubernetes support matrix before installs and upgrades
- [ ] Upgrade with revisions (canary control plane upgrade), not in place
- [ ] Start mTLS in PERMISSIVE, switch to STRICT after all clients have sidecars
- [ ] Default deny AuthorizationPolicy per namespace, allow by ServiceAccount
- [ ] One ServiceAccount per workload
- [ ] Set sidecar CPU and memory requests; use the `Sidecar` CRD to limit config scope in large meshes
- [ ] Run istiod with at least 2 replicas and a PodDisruptionBudget
- [ ] Trace sampling at 1 to 5 percent, spans sent to a persistent backend
- [ ] Automate canary analysis with Argo Rollouts or Flagger
- [ ] Run `istioctl analyze` in CI against manifests before merge
- [ ] Evaluate ambient mode if sidecar overhead is a concern

## 16. Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Pod shows `1/1` | Namespace not labeled, or pod started before the label | `kubectl get ns --show-labels`; restart the deployment |
| `503` with `NR` or no healthy upstream | VirtualService references a subset that has no DestinationRule or no matching pods | `istioctl analyze -n shop`; check pod `version` labels |
| `Connection reset` after STRICT | Client has no sidecar | Add the client to the mesh, or use PERMISSIVE during migration |
| `RBAC: access denied` | No AuthorizationPolicy allows the caller | Check principals and the caller's ServiceAccount |
| Traffic fails with STRICT and a DestinationRule | DestinationRule sets `tls.mode: DISABLE` | Remove it or set `ISTIO_MUTUAL` |
| Only one version gets traffic | Service selector includes `version` | Select only `app` |
| Traces are not joined | App does not forward trace headers | Use the OpenTelemetry SDK |

Useful commands:

```bash
istioctl analyze -n shop                                  # config problems
istioctl proxy-status                                     # are all sidecars in sync with istiod
istioctl proxy-config routes deploy/frontend -n shop      # routes a sidecar actually has
istioctl x describe pod <pod> -n shop                     # which policies and routes apply to a pod
kubectl logs deploy/orders-v1 -n shop -c istio-proxy      # Envoy access logs
```

## 17. Cleanup

```bash
kubectl delete -f manifests/
istioctl uninstall --purge -y
kubectl delete namespace istio-system
```

`--purge` also removes the Istio CRDs, which deletes every Istio resource of those types in the cluster. Do not run it on a shared cluster.

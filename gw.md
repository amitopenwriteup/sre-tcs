# Lab: Canary Rollout with a VirtualService

**Goal:** Take explicit control of traffic splitting across `reviews-v1/v2/v3` using a `DestinationRule` and a weighted `VirtualService`, expose `productpage` through an Istio `Gateway` and the `istio-ingressgateway` service, verify the split actually lands on the sidecar, then shift weights progressively and roll back.

**Prerequisites:**
- Completion of **[Lab: Istio Installation & Sidecar Injection](istio-installation-lab.md)** — this lab assumes Istio is installed and the `bookinfo` namespace is deployed with sidecar injection enabled.
- `bookinfo` namespace has `reviews-v1`, `reviews-v2`, and `reviews-v3` all `2/2 Running`.
- `kubectl` and `istioctl` on your `PATH`.
- The `istio-ingressgateway` service exists in `istio-system` (created by the default Istio install profile).

---

## Background

Right now, traffic to `reviews` round-robins evenly across `v1`, `v2`, and `v3` — that's Istio's default behavior when no `VirtualService` exists for a host. A canary rollout means overriding that default so you control the split explicitly, typically weighting most traffic toward a stable version and a small slice toward a newer one.

To reach `productpage` from outside the cluster the "Istio way," you don't patch the `productpage` Kubernetes `Service` to `NodePort` — you bind a `Gateway` resource to the existing `istio-ingressgateway` Service and route to it with a `VirtualService`. That keeps a single, consistent entry point and lets the same ingress config carry the canary weights all the way from the edge.

---

## Step 1: Define Subsets with a DestinationRule

A `VirtualService` can't route to `v1`/`v2`/`v3` until Istio knows those versions exist as distinct subsets. Define them first:

```yaml
# destination-rule-reviews.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
  namespace: bookinfo
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

Apply it:

```bash
kubectl apply -n bookinfo -f destination-rule-reviews.yaml
kubectl get destinationrule -n bookinfo
```

The `labels` values must match the pod labels on the `reviews-v1/v2/v3` deployments — Bookinfo's sample manifests already set `version: v1|v2|v3`, so no relabeling is needed.

---

## Step 2: Weighted Canary Split with a VirtualService

Route 90% of traffic to the stable `v1` and 10% to the canary `v3`:

```yaml
# virtual-service-reviews-canary.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v3
      weight: 10
```

Apply it:

```bash
kubectl apply -n bookinfo -f virtual-service-reviews-canary.yaml
kubectl get virtualservice -n bookinfo
```

Without this `VirtualService`, Istio's default is even round-robin across all subsets/endpoints of a service — applying it is what actually takes control of the split away from that default.

---

## Step 3: Expose productpage via a Gateway and the ingressgateway Service

Create a `Gateway` that binds to the ingress gateway's workload and opens port 80 for the `bookinfo` host traffic, then a `VirtualService` that routes matching requests to `productpage`:

```yaml
# gateway-bookinfo.yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway   # matches the istio-ingressgateway pod's workload label
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo
  namespace: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

Apply it:

```bash
kubectl apply -n bookinfo -f gateway-bookinfo.yaml
kubectl get gateway -n bookinfo
kubectl get virtualservice -n bookinfo
```

Now find the `istio-ingressgateway` **Service** and how to reach it — this is the actual entry point into the mesh, so you don't need to touch `productpage`'s own `ClusterIP` service or convert it to `NodePort`:

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

- If `TYPE` is `LoadBalancer` and `EXTERNAL-IP` is populated, that IP on port 80 is your entry point:
  ```bash
  export INGRESS_HOST=$(kubectl get svc istio-ingressgateway -n istio-system \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  export INGRESS_PORT=$(kubectl get svc istio-ingressgateway -n istio-system \
    -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
  echo "http://$INGRESS_HOST:$INGRESS_PORT/productpage"
  ```
- If `TYPE` is `NodePort` (e.g. on a bare-metal/kind/minikube cluster with no LoadBalancer support), use a node IP and the assigned NodePort instead:
  ```bash
  export INGRESS_PORT=$(kubectl get svc istio-ingressgateway -n istio-system \
    -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
  export NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
  echo "http://$NODE_IP:$INGRESS_PORT/productpage"
  ```

Either way, `curl` or open that URL — you should reach `productpage` entirely through the mesh ingress, with the `reviews` canary weights from Step 2 still applied behind it.

---

## Step 4: Verify the Split Landed on the Sidecar

Confirm the route actually reached Envoy via xDS, not just the Kubernetes API:

```bash
kubectl get pods -n bookinfo
istioctl proxy-config route <productpage-pod-name> -n bookinfo --name 9080 -o json
```

Look for the `reviews` cluster's weighted clusters (`v1: 90`, `v3: 10`) in the output.

For a live, empirical check, refresh the ingress URL from Step 3 repeatedly (or loop curl) and count how often each version's review styling appears — over ~20 refreshes it should trend roughly 9:1 toward `v1`.

```bash
for i in $(seq 1 20); do
  result=$(kubectl exec -n bookinfo "$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
    -c ratings -- curl -s productpage:9080/productpage | grep -o "color=\"[a-z]*\"" | head -1)
  echo "${result:-v1 (no color)}"
done | sort | uniq -c
```

---

## Step 5: Shift Weights Progressively, Then Roll Back

Increase the canary gradually — e.g., 75/25, then 50/50 — by re-applying the `VirtualService` with updated `weight` values:

```bash
kubectl patch virtualservice reviews -n bookinfo --type merge -p \
  '{"spec":{"http":[{"route":[{"destination":{"host":"reviews","subset":"v1"},"weight":50},{"destination":{"host":"reviews","subset":"v3"},"weight":50}]}]}}'
```

To roll back instantly, route 100% back to the stable subset:

```bash
kubectl patch virtualservice reviews -n bookinfo --type merge -p \
  '{"spec":{"http":[{"route":[{"destination":{"host":"reviews","subset":"v1"},"weight":100}]}]}}'
```

This is the mechanism a progressive-delivery tool (e.g., Flagger or Argo Rollouts) automates — watching metrics and adjusting these same weights — but doing it manually here shows exactly what's being changed underneath.

---

## Cleanup

```bash
kubectl delete virtualservice reviews -n bookinfo
kubectl delete virtualservice bookinfo -n bookinfo
kubectl delete gateway bookinfo-gateway -n bookinfo
kubectl delete destinationrule reviews -n bookinfo
```

If you're also tearing down the base lab environment, follow the Cleanup section in the Istio Installation lab.

---

## Checkpoints (what you should be able to answer after this lab)

- [ ] Why does a `VirtualService` need a `DestinationRule` in place before it can route to specific subsets?
- [ ] What's Istio's default routing behavior across subsets when no `VirtualService` exists?
- [ ] What's the difference between a `Gateway` resource and the `istio-ingressgateway` Kubernetes `Service` — what does each one control?
- [ ] Why is binding a `Gateway` to the `istio-ingressgateway` service preferable to converting `productpage`'s own service to `NodePort`?
- [ ] How would you confirm a canary weight change actually reached Envoy, not just the Kubernetes API?
- [ ] What's the difference between editing the `VirtualService` YAML and using `kubectl patch` to shift weights?
- [ ] How does this manual weight-shifting relate to what a tool like Flagger or Argo Rollouts automates?

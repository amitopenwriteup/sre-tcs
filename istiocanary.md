# Lab: Canary Rollout with a VirtualService

**Goal:** Take explicit control of traffic splitting across `reviews-v1/v2/v3` using a `DestinationRule` and a weighted `VirtualService`, verify the split actually lands on the sidecar, then shift weights progressively and roll back.

**Prerequisites:**
- Completion of **[Lab: Istio Installation & Sidecar Injection](istio-installation-lab.md)** — this lab assumes Istio is installed and the `bookinfo` namespace is deployed with sidecar injection enabled.
- `bookinfo` namespace has `reviews-v1`, `reviews-v2`, and `reviews-v3` all `2/2 Running`.
- `kubectl` and `istioctl` on your `PATH`.

---

## Background

Right now, traffic to `reviews` round-robins evenly across `v1`, `v2`, and `v3` — that's Istio's default behavior when no `VirtualService` exists for a host. A canary rollout means overriding that default so you control the split explicitly, typically weighting most traffic toward a stable version and a small slice toward a newer one.

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

## Step 3: Verify the Split Landed on the Sidecar

Confirm the route actually reached Envoy via xDS, not just the Kubernetes API:

```bash
kubectl get pods -n bookinfo
istioctl proxy-config route <productpage-pod-name> -n bookinfo --name 9080 -o json
```

Look for the `reviews` cluster's weighted clusters (`v1: 90`, `v3: 10`) in the output.
```bash
kubectl edit svc  -n bookinfo
#third last line convert type: NodePort
For a live, empirical check, refresh `http://your node ip:9080/productpage` repeatedly (or loop curl) and count how often each version's review styling appears — over ~20 refreshes it should trend roughly 9:1 toward `v1`.

```bash
for i in $(seq 1 20); do
  kubectl exec -n bookinfo "$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
    -c ratings -- curl -s productpage:9080/productpage | grep -o "color=\"[a-z]*\"" | head -1
done | sort | uniq -c
```

---

## Step 4: Shift Weights Progressively, Then Roll Back

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
kubectl delete destinationrule reviews -n bookinfo
```

If you're also tearing down the base lab environment, follow the Cleanup section in the Istio Installation lab.

---

## Checkpoints (what you should be able to answer after this lab)

- [ ] Why does a `VirtualService` need a `DestinationRule` in place before it can route to specific subsets?
- [ ] What's Istio's default routing behavior across subsets when no `VirtualService` exists?
- [ ] How would you confirm a canary weight change actually reached Envoy, not just the Kubernetes API?
- [ ] What's the difference between editing the `VirtualService` YAML and using `kubectl patch` to shift weights?
- [ ] How does this manual weight-shifting relate to what a tool like Flagger or Argo Rollouts automates?

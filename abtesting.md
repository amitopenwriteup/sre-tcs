# Lab: A/B Testing with a VirtualService

**Goal:** Route traffic to `reviews-v1/v2` based on *user identity* (a header, cookie, or query param) rather than random weight — so the same user consistently lands on the same version — then verify the match actually lands on the sidecar, and roll back.

**Prerequisites:**
- Completion of **[Lab: Istio Installation & Sidecar Injection](istio-installation-lab.md)** — this lab assumes Istio is installed and the `bookinfo` namespace is deployed with sidecar injection enabled.
- `bookinfo` namespace has `reviews-v1` and `reviews-v2` both `2/2 Running`.
- `kubectl` and `istioctl` on your `PATH`.
- Helpful but not required: completion of the **Canary Rollout** lab, since this one reuses the same `DestinationRule` concept and contrasts directly with weighted routing.

---

## Background

Canary rollout and A/B testing are often confused because both use a `VirtualService` to split traffic across subsets — but they solve different problems:

| | Canary | A/B testing |
|---|---|---|
| Split basis | Random weight (`weight: 90/10`) | User identity (header, cookie, query param) |
| Goal | De-risk a rollout | Compare user behavior/metrics between variants |
| Same user, repeat request | May hit either version | Should consistently hit the same version |

A/B testing needs `match` conditions in the `VirtualService` `http` routes instead of (or alongside) `weight`. Istio evaluates `http` route rules top-to-bottom and uses the first one whose `match` conditions are satisfied, falling through to a default route with no `match` if none match.

---

## Step 1: Define Subsets with a DestinationRule

Same as the canary lab — a `VirtualService` can't route to `v1`/`v2` until Istio knows those versions exist as distinct subsets.

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
```

Apply it:

```bash
kubectl apply -n bookinfo -f destination-rule-reviews.yaml
kubectl get destinationrule -n bookinfo
```

If you already applied the `DestinationRule` from the canary lab with a `v3` subset included, that's fine — this lab only uses `v1` and `v2`, the extra subset is simply unused.

---

## Step 2: Identity-Based Split with a VirtualService

Route requests carrying a specific header to `v2` (the "B" variant), and send everyone else to `v1` (the "A" / control variant):

```yaml
# virtual-service-reviews-ab.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: variant-b
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

Apply it:

```bash
kubectl apply -n bookinfo -f virtual-service-reviews-ab.yaml
kubectl get virtualservice -n bookinfo
```

Notes:
- The first `http` entry only matches when the `end-user` header is exactly `variant-b`. Everything else falls through to the second entry, which has no `match` and acts as the default.
- In a real deployment, this header is usually set by an authentication layer or edge proxy based on a stable user attribute (user ID hash, cookie, logged-in segment) — not sent raw by the client — so the same user is deterministically bucketed on every request.
- Order matters: the matched rule must come before the unconditional default, or the default will shadow it.

---

## Step 3: Verify the Split Landed on the Sidecar

Confirm the route actually reached Envoy via xDS, not just the Kubernetes API:

```bash
kubectl get pods -n bookinfo
istioctl proxy-config route <productpage-pod-name> -n bookinfo --name 9080 -o json
```

Look for two entries under the `reviews` route: one with a header match on `end-user: variant-b` routing to the `v2` subset, and a fallback routing to `v1`.

For a live, empirical check, send requests with and without the header and confirm each consistently lands on the expected version:

```bash
POD=$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')

echo "Without header (expect v1, repeatedly):"
for i in $(seq 1 5); do
  kubectl exec -n bookinfo "$POD" -c ratings -- \
    curl -s productpage:9080/productpage | grep -o "color=\"[a-z]*\"" | head -1
done

echo "With header end-user: variant-b (expect v2, repeatedly):"
for i in $(seq 1 5); do
  kubectl exec -n bookinfo "$POD" -c ratings -- \
    curl -s -H "end-user: variant-b" productpage:9080/productpage | grep -o "color=\"[a-z]*\"" | head -1
done
```

Unlike the canary lab's ~9:1 trend across random refreshes, here each group should be **100% consistent** within itself — that consistency is the whole point of identity-based routing.

---

## Step 4: Add a Second Variant Criterion, Then Roll Back

You can match on more than a static header — query params and cookies are common for real A/B tests where you don't control request headers directly:

```bash
kubectl patch virtualservice reviews -n bookinfo --type merge -p \
  '{"spec":{"http":[
    {"match":[{"queryParams":{"variant":{"exact":"b"}}}],"route":[{"destination":{"host":"reviews","subset":"v2"}}]},
    {"route":[{"destination":{"host":"reviews","subset":"v1"}}]}
  ]}}'
```

This routes `?variant=b` to `v2`, everything else to `v1`.

To roll back and send all traffic to the control variant regardless of any header or param:

```bash
kubectl patch virtualservice reviews -n bookinfo --type merge -p \
  '{"spec":{"http":[{"route":[{"destination":{"host":"reviews","subset":"v1"}}]}]}}'
```

This is the mechanism behind most experimentation platforms (e.g. LaunchDarkly, Optimizely feature-flag-driven routing, or a service mesh-aware A/B tool) — they manage the bucketing logic and consistently set an identifying header or cookie, then let Istio's `match` rules do the actual routing.

---

## Cleanup

```bash
kubectl delete virtualservice reviews -n bookinfo
kubectl delete destinationrule reviews -n bookinfo
```

If you're also tearing down the base lab environment, follow the Cleanup section in the Istio Installation lab.

---

## Checkpoints (what you should be able to answer after this lab)

- [ ] What's the core difference between canary routing and A/B testing routing, given both use a `VirtualService`?
- [ ] Why does `match`-based routing require a *deterministic* signal (header, cookie) rather than something the client can change between requests?
- [ ] Why must the matched `http` route entry appear before the unconditional default entry?
- [ ] How would you confirm a match rule actually reached Envoy, not just the Kubernetes API?
- [ ] Where would the `end-user` (or similar) header realistically get set in a production request path?

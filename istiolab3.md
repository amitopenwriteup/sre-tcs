## Step 5: Add a Match Expression for Targeted Canary Testing

**Goal:** Combine a `match` expression with the weighted split — route specific requests (e.g., internal testers) straight to the canary `v3` every time, while everyone else still gets the probabilistic 90/10 split from Step 2.

### Add a Header Match Rule Above the Weighted Split

Route any request carrying the header `end-user: jason` straight to `v3`, and leave everyone else on the existing 90/10 weighted split:

```yaml
# virtual-service-reviews-canary-match.yaml
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
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v3
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
kubectl apply -n bookinfo -f virtual-service-reviews-canary-match.yaml
kubectl get virtualservice reviews -n bookinfo -o yaml
```

**Read the two rules:**

| Rule | Condition | Destination |
|---|---|---|
| 1 | request header `end-user` exactly equals `jason` | 100% → `v3` |
| 2 | (no match — catches everything else) | weighted 90% `v1` / 10% `v3` |

### Verify on the Sidecar

```bash
istioctl proxy-config route <productpage-pod-name> -n bookinfo --name 9080 -o json
```

Confirm the header match condition and its unconditional route to `v3` appear as a **separate, earlier** route entry from the weighted `v1`/`v3` split.

### Test It

Requests carrying the header should land on `v3` every single time — deterministic, not a ratio:

```bash
RATINGS_POD=$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')

for i in $(seq 1 10); do
  kubectl exec -n bookinfo "$RATINGS_POD" -c ratings -- \
    curl -s -H "end-user: jason" reviews:9080/reviews/1 | grep -oE "\"color\": ?\"[a-z]*\""
done | sort | uniq -c
```

**Expected output:** all 10 requests show `"color":"red"` (or whatever `v3`'s rating color is) — since every request in this loop carries the matching header, none of them ever reach the weighted rule.

Now drop the header and confirm it falls back to the weighted split from Step 2:

```bash
for i in $(seq 1 20); do
  kubectl exec -n bookinfo "$RATINGS_POD" -c ratings -- \
    curl -s reviews:9080/reviews/1 | grep -o "\"color\":\"[a-z]*\""
  echo "${result:-v1 (no color)}"
done | sort | uniq -c
```

**Expected output:** a rough 9:1 split between `v1 (no color)` and `v3`'s color, matching Step 3's weighted-split result — confirming the matched rule only intercepts requests that actually carry the header, and everything else still flows through the weighted logic underneath.

### Restore the Plain Weighted Rule

```bash
kubectl apply -n bookinfo -f virtual-service-reviews-canary.yaml
```

---

## Updated Cleanup

```bash
kubectl delete virtualservice reviews -n bookinfo
kubectl delete destinationrule reviews -n bookinfo
```

If you're also tearing down the base lab environment, follow the Cleanup section in the Istio Installation lab.

---

## Updated Checkpoints

- [ ] Why does a `VirtualService` need a `DestinationRule` in place before it can route to specific subsets?
- [ ] What's Istio's default routing behavior across subsets when no `VirtualService` exists?
- [ ] How would you confirm a canary weight change actually reached Envoy, not just the Kubernetes API?
- [ ] What's the difference between editing the `VirtualService` YAML and using `kubectl patch` to shift weights?
- [ ] How does this manual weight-shifting relate to what a tool like Flagger or Argo Rollouts automates?
- [ ] Why does placing a `match` rule *above* a weighted rule let it intercept traffic before the weighted logic runs?
- [ ] What's the practical difference between testing a matched rule (Step 5) and testing a weighted rule (Step 3) — why does one need only a handful of calls while the other needs a larger sample?
- [ ] What other `match` conditions (besides headers) could you use to target specific traffic at a canary — and when would each make sense?

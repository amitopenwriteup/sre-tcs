## Step 5: Regex Match — Route Beta-Testers via a Cookie

**Goal:** Use a `regex` match (instead of `exact` or `prefix`) to detect a `canary=true` flag anywhere inside the `Cookie` header, and route those requests straight to `v3` — while everyone else stays on the existing weighted split from Step 2.

### Add a Regex Cookie Match Rule

```yaml
# virtual-service-reviews-regex.yaml
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
        cookie:
          regex: "^(.*;\\s*)?canary=true(;.*)?$"
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
kubectl apply -n bookinfo -f virtual-service-reviews-regex.yaml
kubectl get virtualservice reviews -n bookinfo -o yaml
```

**Read the two rules:**

| Rule | Condition | Match type | Destination |
|---|---|---|---|
| 1 | `Cookie` header contains `canary=true` anywhere among its `;`-separated values | `regex` | 100% → `v3` |
| 2 | (no match — catches everything else) | — | weighted 90% `v1` / 10% `v3` |

A cookie header can carry several `key=value` pairs separated by `; `, so an `exact` match wouldn't reliably catch `canary=true` if other cookies are present before or after it — that's exactly the case a `regex` match is built for. `regex` uses RE2 syntax (the same engine Envoy uses internally).

### Verify on the Sidecar

```bash
istioctl proxy-config route <productpage-pod-name> -n bookinfo --name 9080 -o json
```

Confirm the `cookie` regex match appears as its own route entry, positioned above the weighted fallback.

### Test It

```bash
RATINGS_POD=$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')

# Cookie carries canary=true alongside another cookie -> should always hit v3
for i in $(seq 1 10); do
  kubectl exec -n bookinfo "$RATINGS_POD" -c ratings -- \
    curl -s -H "Cookie: session_id=abc123; canary=true" reviews:9080/reviews/1 \
    | grep -o "\"color\":\"[a-z]*\""
done | sort | uniq -c

# No canary cookie -> falls through to the weighted rule
for i in $(seq 1 20); do
  result=$(kubectl exec -n bookinfo "$RATINGS_POD" -c ratings -- \
    curl -s -H "Cookie: session_id=abc123" reviews:9080/reviews/1 \
    | grep -o "\"color\":\"[a-z]*\"")
  echo "${result:-v1 (no color)}"
done | sort | uniq -c
```

**Expected result:** the `canary=true` requests land on `v3`'s color every time (10/10), regardless of what other cookies surround it. The plain-session requests skip the matched rule and fall through to the weighted split, landing roughly 9:1 on `v1 (no color)` vs. `v3`'s color — the same distribution as Step 3.

### Restore the Prior Rule

```bash
kubectl apply -n bookinfo -f virtual-service-reviews-canary.yaml
```

---

## Updated Checkpoints

- [ ] What's the difference between `exact`, `prefix`, and `regex` match types on a header condition?
- [ ] Why does a cookie header often need a `regex` match instead of `exact`, given how cookies are formatted?
- [ ] Why does rule order matter when a `regex` match rule and a weighted fallback both target the same host?
- [ ] What regex engine does Envoy (and therefore Istio) use, and how does that affect the syntax you can rely on?

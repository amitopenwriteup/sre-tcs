# Lab: URI Matching & Rewrite Routing with a VirtualService

**Goal:** Route traffic to a specific service subset based on **request path** (not weight), rewrite that path before it's forwarded upstream, and verify the match/rewrite actually took effect on the sidecar — as distinct from weighted/canary routing.

**Prerequisites:**
- Completion of **[Lab: Istio Installation & Sidecar Injection](istio-installation-lab.md)** — this lab assumes Istio is installed and the `bookinfo` namespace is deployed with sidecar injection enabled.
- `bookinfo` namespace has `reviews-v1`, `reviews-v2`, and `reviews-v3` all `2/2 Running`.
- `kubectl` and `istioctl` on your `PATH`.

---

## Background

A `VirtualService` `http` route list is evaluated **top to bottom**, and Envoy stops at the **first matching rule**. Each rule can optionally include a `match` block:

- **With a `match` block** — the rule only applies to requests meeting the condition(s) inside it (e.g., a URI prefix). Multiple entries inside a single `match` block are combined with **OR** logic, not AND.
- **Without a `match` block** — the rule applies to everything, so it only makes sense as the last rule in the list (a catch-all/default).

A `rewrite.uri` field, when present on a matched rule, replaces the request path **before** it's sent to the upstream destination. The client never sees the rewritten path — only the upstream service does.

This is different from **weighted routing**, where every request has a *probability* of landing on each subset regardless of what the request contains. Path-based routing is **deterministic**: the same path always produces the same routing decision. That distinction matters for how you test it — a weighted rule is verified by sampling many requests and checking the ratio; a path-matched rule is verified by checking that one specific path *always* goes to one specific place.

---

## Step 1: Define Subsets with a DestinationRule

Before any `VirtualService` can route to named subsets like `v1` or `v2`, Istio needs a `DestinationRule` declaring which pod labels each subset name refers to.

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
kubectl get destinationrule reviews -n bookinfo -o yaml
```

`v1` and `v2` here map to the `version: v1` / `version: v2` labels already set on Bookinfo's `reviews-v1` and `reviews-v2` deployments — no relabeling needed.

---

## Step 2: Write a URI-Matched Rewrite VirtualService

Route requests under `/wpcatalog` or `/consumercatalog` to `v2`, rewriting the path to `/newcatalog` first. Everything else falls through to `v1`, unchanged.

```yaml
# virtual-service-reviews-rewrite.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews # interpreted as reviews.bookinfo.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
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
kubectl apply -n bookinfo -f virtual-service-reviews-rewrite.yaml
kubectl get virtualservice reviews -n bookinfo -o yaml
```

**Read the two rules carefully:**

| Rule | Condition | Rewrite | Destination |
|---|---|---|---|
| 1 | path starts with `/wpcatalog` **or** `/consumercatalog` | → `/newcatalog` | `reviews` subset `v2` |
| 2 | (no match block — catches everything else) | none | `reviews` subset `v1` |

---

## Step 3: Verify the Rule Landed on the Sidecar

Config applied via `kubectl` isn't automatically proof that Envoy received it. Confirm via `istioctl`:

```bash
kubectl get pods -n bookinfo
istioctl proxy-config route <productpage-pod-name> -n bookinfo --name 9080 -o json
```

In the output, find the `reviews` route entries and confirm:
- A prefix match on `/wpcatalog` and one on `/consumercatalog`
- A `path_rewrite` (or equivalent rewrite field) set to `/newcatalog`
- The matched route pointing at the `v2` cluster
- A separate, unconditional route pointing at the `v1` cluster

---

## Step 4: Test the Routing Directly Against `reviews`

Unlike a weighted split, this rule should be **100% consistent per path** — there's no ratio to sample, only a yes/no per path. Call `reviews` directly (not `productpage`, which only ever calls `/reviews/{id}` and would never exercise the match rule), and confirm via the sidecar access logs which subset handled each request.

```bash
RATINGS_POD=$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')

# Matches rule 1 -> rewritten -> v2
kubectl exec -n bookinfo "$RATINGS_POD" -c ratings -- curl -s -o /dev/null -w "%{http_code}\n" reviews:9080/wpcatalog/1
kubectl exec -n bookinfo "$RATINGS_POD" -c ratings -- curl -s -o /dev/null -w "%{http_code}\n" reviews:9080/consumercatalog/1

# Matches rule 2 (no match block) -> unchanged -> v1
kubectl exec -n bookinfo "$RATINGS_POD" -c ratings -- curl -s -o /dev/null -w "%{http_code}\n" reviews:9080/reviews/1
```

Check the sidecar access logs to see which subset actually served each request, and confirm the rewritten path:

```bash
kubectl logs -n bookinfo -l app=reviews,version=v2 -c istio-proxy --tail=10 | grep -i newcatalog
kubectl logs -n bookinfo -l app=reviews,version=v1 -c istio-proxy --tail=10 | grep -i "GET /reviews"
```

**Expected result:** every `/wpcatalog/*` and `/consumercatalog/*` call appears in the **v2** log as `GET /newcatalog/...`; every `/reviews/*` call appears in the **v1** log unchanged.

> **Note:** the stock Bookinfo `reviews` app only implements `/reviews/{id}` — it has no `/newcatalog` route, so `v2` will return an HTTP 404 **body** even when the routing/rewrite itself worked correctly. The `curl -w "%{http_code}"` and access-log checks above verify the *routing decision*, which is independent of whether the upstream app happens to have a matching endpoint. A 404 with the request landing in the `v2` log is a **pass**, not a failure, for this lab.

### Repeatability check (proving it's deterministic, not probabilistic)

```bash
for i in $(seq 1 10); do
  kubectl exec -n bookinfo "$RATINGS_POD" -c ratings -- curl -s -o /dev/null -w "%{http_code}\n" reviews:9080/wpcatalog/1
done | sort | uniq -c
```

Every one of the 10 calls should return the **same** status code — if you see a mix, something's off (stale DestinationRule, mislabeled pods, or a competing `VirtualService` for the same host).

---

## Step 5: Remove or Widen the Match

To see the effect of the match condition itself, try narrowing or removing it and re-testing:

```bash
kubectl patch virtualservice reviews -n bookinfo --type merge -p \
  '{"spec":{"http":[{"match":[{"uri":{"prefix":"/wpcatalog"}}],"rewrite":{"uri":"/newcatalog"},"route":[{"destination":{"host":"reviews","subset":"v2"}}]},{"route":[{"destination":{"host":"reviews","subset":"v1"}}]}]}}'
```

This drops `/consumercatalog` from the match. Re-run Step 4's calls — `/consumercatalog/1` should now fall through to `v1` instead of `v2`, since it no longer matches rule 1.

Restore the full two-prefix version:

```bash
kubectl apply -n bookinfo -f virtual-service-reviews-rewrite.yaml
```

---

## Cleanup

```bash
kubectl delete virtualservice reviews -n bookinfo
kubectl delete destinationrule reviews -n bookinfo
```

If you're also tearing down the base lab environment, follow the Cleanup section in the Istio Installation lab.

---

## Checkpoints (what you should be able to answer after this lab)

- [ ] What does `match.uri.prefix` actually test against, and what happens if a rule has no `match` block at all?
- [ ] Why is a `match` block with two `uri` entries an OR condition, not an AND?
- [ ] What does `rewrite.uri` change, and does the original client ever see the rewritten path?
- [ ] Why does a `VirtualService` still need a `DestinationRule` in place before it can route to `subset: v2`?
- [ ] Why is a repeated-sample test (`for i in seq...`) the right tool to verify a *weighted* rule but a poor way to reason about a *path-matched* rule?
- [ ] If `reviews:9080/wpcatalog/1` returns a 404 body, how do you tell whether that's a routing failure or an application-level "no such endpoint" — and why does that distinction matter?
- [ ] What real-world migration scenario does the "match old path, rewrite to new path, route to new subset" pattern typically solve?

# Lab: Istio URI Rewrite & Subset Routing (`reviews-route`)

Using the existing `bookinfo` namespace deployment (`reviews-v1`, `reviews-v2`, `reviews-v3` already running).

## 1. VirtualService

```yaml
# reviews-route.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews-route
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
        host: reviews # interpreted as reviews.bookinfo.svc.cluster.local
        subset: v2
  - route:
    - destination:
        host: reviews # interpreted as reviews.bookinfo.svc.cluster.local
        subset: v1
```

```bash
kubectl apply -f reviews-route.yaml -n bookinfo
```

## 2. DestinationRule

```yaml
# reviews-destination.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews-destination
  namespace: bookinfo
spec:
  host: reviews # interpreted as reviews.bookinfo.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

```bash
kubectl apply -f reviews-destination.yaml -n bookinfo
```

## 3. Test

```bash
kubectl exec -n bookinfo "$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
  -c ratings -- curl -sv http://reviews:9080/wpcatalog/1 2>&1 | grep -i "< HTTP"

kubectl exec -n bookinfo "$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
  -c ratings -- curl -sv http://reviews:9080/consumercatalog/1 2>&1 | grep -i "< HTTP"

kubectl exec -n bookinfo "$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
  -c ratings -- curl -sv http://reviews:9080/reviews/1 2>&1 | grep -i "< HTTP"
```

Confirm which subset served each path via sidecar access logs:

```bash
kubectl logs -n bookinfo -l app=reviews,version=v2 -c istio-proxy --tail=10 | grep -i newcatalog
kubectl logs -n bookinfo -l app=reviews,version=v1 -c istio-proxy --tail=10 | grep -i "GET /reviews"
```

**Expected result:** `/wpcatalog/1` and `/consumercatalog/1` are rewritten to `/newcatalog/1` and appear in the **v2** access log; `/reviews/1` appears unchanged in the **v1** access log.

### Color/version tally over 20 requests

```bash
for i in $(seq 1 20); do
  result=$(kubectl exec -n bookinfo "$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
    -c ratings -- curl -s reviews:9080/reviews/1 | grep -o "\"color\":\"[a-z]*\"" | head -1)
  echo "${result:-v1 (no color)}"
done | sort | uniq -c
```

**Expected output:**

```
     20 v1 (no color)
```

`/reviews/1` doesn't match `/wpcatalog` or `/consumercatalog`, so it always falls through to the default rule → **subset v1**. `v1` never calls the `ratings` service, so its JSON response has no `rating.color` field — hence a consistent 20/20 result.

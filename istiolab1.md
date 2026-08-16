# Lab: Istio VirtualService URI Rewrite & Subset Routing

## Objective

Learn how Istio's `VirtualService` and `DestinationRule` work together to:
- Match incoming requests by URI prefix
- Rewrite the URI before forwarding
- Route traffic to specific service **subsets** (versions) based on Kubernetes pod labels
- Fall back to a default subset when no match occurs

---

## 1. Concepts

| Resource | Role |
|---|---|
| `VirtualService` | Defines **routing rules** — how requests to a host are matched, rewritten, and directed to a destination/subset |
| `DestinationRule` | Defines **subsets** — named groupings of pods for a host, selected by label |

Istio evaluates `VirtualService` HTTP rules **top to bottom**, and stops at the **first match**. A rule with no `match` block acts as the catch-all/default.

---

## 2. Manifests

### 2.1 VirtualService

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews-route
  namespace: foo
spec:
  hosts:
  - reviews # interpreted as reviews.foo.svc.cluster.local
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
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        subset: v2
  - route:
    - destination:
        host: reviews # interpreted as reviews.foo.svc.cluster.local
        subset: v1
```

### 2.2 DestinationRule

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews-destination
  namespace: foo
spec:
  host: reviews # interpreted as reviews.foo.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

---

## 3. Explanation

### VirtualService (`reviews-route`)

**Rule 1 — matched first**
```yaml
match:
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
```
- Multiple entries inside one `match` block are **OR** conditions, not AND. So this matches any request whose path starts with `/wpcatalog` **or** `/consumercatalog`.
- The `rewrite.uri` field replaces the path with `/newcatalog` before the request is forwarded upstream.
- Matching requests are sent to the `v2` subset.

**Rule 2 — fallback**
```yaml
route:
- destination:
    host: reviews
    subset: v1
```
- No `match` block means this rule catches everything that didn't match Rule 1.
- These requests are forwarded unchanged to the `v1` subset.

### DestinationRule (`reviews-destination`)

Defines what the subset names `v1` and `v2` actually resolve to — pods matched by Kubernetes labels:

- `v1` → pods with label `version: v1`
- `v2` → pods with label `version: v2`

The `VirtualService` references these subset **names**; the `DestinationRule` supplies the **label selectors** behind them. Both resources must exist and target the same `host` for routing to work.

### Combined behavior

| Incoming path | Rewritten path | Destination subset | Pods selected |
|---|---|---|---|
| `/wpcatalog/*` | `/newcatalog/*` | `v2` | `version: v2` |
| `/consumercatalog/*` | `/newcatalog/*` | `v2` | `version: v2` |
| anything else | unchanged | `v1` | `version: v1` |

This is a common pattern for **migrating a deprecated API path**: old paths are transparently rewritten and served by the new service version, while all other traffic stays on the stable version.

---

## 4. Hands-On Lab

### Prerequisites
- A Kubernetes cluster with Istio installed (`istioctl install` or equivalent)
- `kubectl` configured against the cluster
- Sidecar injection enabled on the target namespace

### Step 1 — Create the namespace and enable injection

```bash
kubectl create namespace foo
kubectl label namespace foo istio-injection=enabled
```

### Step 2 — Deploy two versions of the `reviews` service

```yaml
# reviews-v1-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v1
  namespace: foo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
      version: v1
  template:
    metadata:
      labels:
        app: reviews
        version: v1
    spec:
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
        ports:
        - containerPort: 9080
---
# reviews-v2-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v2
  namespace: foo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
      version: v2
  template:
    metadata:
      labels:
        app: reviews
        version: v2
    spec:
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v2:1.20.2
        ports:
        - containerPort: 9080
---
# reviews-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: reviews
  namespace: foo
spec:
  selector:
    app: reviews
  ports:
  - port: 9080
    name: http
```

```bash
kubectl apply -f reviews-v1-deployment.yaml
kubectl apply -f reviews-v2-deployment.yaml
kubectl apply -f reviews-service.yaml
```

### Step 3 — Apply the DestinationRule

Save the DestinationRule from section 2.2 as `reviews-destination.yaml`:

```bash
kubectl apply -f reviews-destination.yaml
```

Verify:
```bash
kubectl get destinationrule reviews-destination -n foo -o yaml
```

### Step 4 — Apply the VirtualService

Save the VirtualService from section 2.1 as `reviews-route.yaml`:

```bash
kubectl apply -f reviews-route.yaml
```

Verify:
```bash
kubectl get virtualservice reviews-route -n foo -o yaml
```

### Step 5 — Test the routing

Exec into a pod with `curl` inside the mesh (or use a sleep/curl test pod):

```bash
kubectl run curl-test --image=curlimages/curl -n foo -it --rm -- sh
```

From inside the pod, test each path:

```bash
# Should be rewritten to /newcatalog and hit v2
curl -v http://reviews.foo.svc.cluster.local:9080/wpcatalog/123

# Should be rewritten to /newcatalog and hit v2
curl -v http://reviews.foo.svc.cluster.local:9080/consumercatalog/456

# No match on rule 1 -> falls through to v1, no rewrite
curl -v http://reviews.foo.svc.cluster.local:9080/anything-else
```

### Step 6 — Confirm which subset served the request

Check logs on each deployment to confirm which version received traffic:

```bash
kubectl logs -l app=reviews,version=v1 -n foo --tail=20
kubectl logs -l app=reviews,version=v2 -n foo --tail=20
```

You should see:
- Requests to `/wpcatalog` and `/consumercatalog` (rewritten to `/newcatalog`) appearing in the **v2** pod logs
- Requests to any other path appearing in the **v1** pod logs

### Step 7 (optional) — Inspect Envoy route config directly

```bash
istioctl proxy-config routes <curl-test-pod> -n foo -o json
```

Look for the `reviews` route entries and confirm the prefix matches, rewrite rule, and cluster (subset) weighting match what's defined in the `VirtualService`.

### Cleanup

```bash
kubectl delete -f reviews-route.yaml
kubectl delete -f reviews-destination.yaml
kubectl delete -f reviews-service.yaml
kubectl delete -f reviews-v1-deployment.yaml
kubectl delete -f reviews-v2-deployment.yaml
kubectl delete namespace foo
```

---

## 5. Key Takeaways

- `VirtualService` rules are evaluated **in order**; the first matching rule wins.
- A `match` block with multiple `uri` entries is an **OR**, not an AND.
- `rewrite.uri` changes the path sent to the upstream service — the client never sees the rewritten path.
- `DestinationRule` subsets are just label selectors; they don't route traffic on their own — a `VirtualService` (or the mesh's default routing) must reference them.
- This pattern (`match` + `rewrite` + `subset`) is the standard way to migrate deprecated API paths to a new service version without breaking old clients.

---

*Note: `reviews`, `wpcatalog`, and `consumercatalog` are the canonical example names used in Istio's official documentation, not a real production service.*

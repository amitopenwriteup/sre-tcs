# Istio Troubleshooting Workshop Lab

**Goal:** Master Istio diagnostics by simulating real failures: sidecar injection issues, 503 errors, connection problems, and mTLS failures. Learn to use `istioctl` and Envoy inspection tools to fix them.

**Prerequisites:**
- Kubernetes cluster (minikube, kind, or managed)
- **Istio 1.14+** installed with sidecar injection enabled
- `kubectl` and `istioctl` on your `PATH`
- `bookinfo` sample app deployed, or willingness to deploy it
- Basic understanding of Kubernetes Deployments, Services, and Namespaces

**Estimated Time:** 3-4 hours

**Cluster Check:**

```bash
istioctl version
kubectl get ns -l istio-injection=enabled
kubectl get crd | grep istio
```

---

## Part 1: Sidecar Injection Failures

### Problem: Pods deploy without Envoy sidecar, so traffic doesn't route through Istio

The Envoy sidecar is the core of Istio's traffic management. Without it, VirtualServices and DestinationRules are ignored.

### Section 1.1: Understand Sidecar Injection

Sidecar injection happens automatically when a pod is created in a namespace labeled `istio-injection=enabled`. The Istio MutatingWebhookConfiguration intercepts pod creation and injects the sidecar.

Check which namespaces have injection enabled:

```bash
kubectl get ns -L istio-injection
```

Output:
```
NAME            STATUS   AGE    ISTIO-INJECTION
default         Active   10d    disabled
istio-system    Active   10d    enabled
kube-system     Active   10d    
bookinfo        Active   2d     enabled
troubleshooting Active   1d     disabled
```

### Section 1.2: Simulate Sidecar Injection Failure

Create a namespace WITHOUT injection enabled:

```bash
kubectl create namespace inject-lab
# Notice: no label added
```

Deploy an app:

```yaml
# app-no-injection.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: inject-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:latest
        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f app-no-injection.yaml
kubectl get pods -n inject-lab -o jsonpath='{.items[0].spec.containers[*].name}'
```

Output:
```
web
```

Only one container! The sidecar was NOT injected. With injection enabled, you'd see:
```
web envoy
```

### Section 1.3: Diagnose Sidecar Injection Failures

**Check 1: Is the namespace labeled?**

```bash
kubectl get ns inject-lab -o yaml | grep istio-injection
# No output = not labeled
```

**Check 2: Is the MutatingWebhookConfiguration active?**

```bash
kubectl get mutatingwebhookconfigurations | grep istio
```

Output:
```
istio-sidecar-injector   1            15d
```

Get details:

```bash
kubectl describe mutatingwebhookconfigurations istio-sidecar-injector
```

Look for:
- `Rules:` section — should match pods in labeled namespaces
- `FailurePolicy: Ignore` — what happens if webhook fails?
- `reinvocationPolicy: Never` — prevents re-applying webhook

**Check 3: Is the webhook receiving requests?**

Check the mutating webhook configuration's label selector:

```bash
kubectl get mutatingwebhookconfigurations istio-sidecar-injector -o yaml | \
  grep -A 5 "namespaceSelector:"
```

Output:
```yaml
namespaceSelector:
  matchLabels:
    istio-injection: enabled
```

The webhook ONLY fires on namespaces with label `istio-injection: enabled`.

**Check 4: Verify the webhook service exists**

```bash
kubectl get svc -n istio-system | grep sidecar-injector
```

Output:
```
istiod   ClusterIP   10.96.190.10   <none>   443/TCP,...
```

The webhook service must be accessible from the API server.

**Check 5: Look for webhook errors in logs**

```bash
kubectl logs -n istio-system -l app=istiod | grep -i "webhook\|inject"
```

### Section 1.4: Fix Sidecar Injection Failures

**Fix 1: Enable Namespace Label (Recommended)**

```bash
kubectl label namespace inject-lab istio-injection=enabled
kubectl rollout restart deployment web -n inject-lab
kubectl get pods -n inject-lab -o jsonpath='{.items[0].spec.containers[*].name}'
```

Output:
```
web envoy
```

Now the sidecar is injected!

**Verify sidecar is running:**

```bash
kubectl describe pod web-<pod-id> -n inject-lab | grep -A 10 Containers:
```

Output:
```
Containers:
  web:
    Container ID:   ...
    Image:          nginx:latest
  envoy:
    Container ID:   ...
    Image:          proxyv2:1.17.0 (or current version)
```

**Fix 2: Manual Sidecar Injection (if webhook fails)**

If the webhook is broken, manually inject:

```bash
kubectl set env deployment/web ISTIO_INJECTION=enabled -n inject-lab
# This won't actually inject — you need to use istioctl manually:

istioctl kube-inject -f app-no-injection.yaml | kubectl apply -f -
```

The `istioctl kube-inject` command adds the sidecar container to your YAML, then you apply it.

**Fix 3: Fix Webhook Configuration**

If the webhook service is down, restart it:

```bash
kubectl rollout restart deployment istiod -n istio-system
```

Check webhook logs:

```bash
kubectl logs -n istio-system -l app=istiod -f
# Wait for "webhook server started" message
```

### Section 1.5: Verify Sidecar Is Working

Once injected, verify Envoy is processing traffic:

```bash
# Create a service
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: inject-lab
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
EOF

# Create a VirtualService
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: web
  namespace: inject-lab
spec:
  hosts:
  - web
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: web
        port:
          number: 80
EOF

# Create a client pod
kubectl run client --image=curlimages/curl -n inject-lab -- sleep 3600

# Test: does traffic go through Envoy?
kubectl exec -it client -n inject-lab -- curl http://web
# Should succeed and return nginx welcome page
```

If it fails, the sidecar likely isn't configured properly.

### Section 1.6: Sidecar Injection Troubleshooting Checklist

| Issue | Command | Fix |
|-------|---------|-----|
| Namespace not labeled | `kubectl get ns -L istio-injection` | `kubectl label namespace <ns> istio-injection=enabled` |
| Webhook not firing | `kubectl get mutatingwebhookconfigurations istio-sidecar-injector` | Restart istiod: `kubectl rollout restart -n istio-system deployment/istiod` |
| Old pods not injected | Check pod age | Restart deployment: `kubectl rollout restart deployment/<name>` |
| Webhook service down | `kubectl get svc -n istio-system \| grep injector` | Restart istiod |
| Sidecar container not starting | `kubectl logs <pod> -c envoy` | Check Envoy logs for startup errors |
| VirtualService has no effect | Verify sidecar injected: `kubectl get pod <pod> -o yaml \| grep envoy` | Inject sidecar, restart pod |

---

## Part 2: HTTP 503 Service Unavailable

### Problem: Requests return 503, indicating the backend is unavailable to Envoy

503 usually means Envoy can't connect to the upstream service. Causes:
- No healthy endpoints backing the service
- Outlier detection evicting all endpoints
- Circuit breaker rules rejecting all connections
- Service misconfigured in Istio

### Section 2.1: Simulate 503 Error

Create two services: one healthy, one broken.

```yaml
# services-503-scenario.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: error-lab
  labels:
    istio-injection: enabled

---
# Healthy backend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: healthy-backend
  namespace: error-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: healthy-backend
  template:
    metadata:
      labels:
        app: healthy-backend
    spec:
      containers:
      - name: app
        image: httpbin:latest
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: healthy-backend
  namespace: error-lab
spec:
  selector:
    app: healthy-backend
  ports:
  - port: 80
    targetPort: 80

---
# Broken backend (no pods backing it)
apiVersion: v1
kind: Service
metadata:
  name: broken-backend
  namespace: error-lab
spec:
  selector:
    app: nonexistent-app  # No pods match this label
  ports:
  - port: 80
    targetPort: 80

---
# Client
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: error-lab
spec:
  containers:
  - name: curl
    image: curlimages/curl
    command: ["sleep", "3600"]

---
# VirtualService routing to broken backend
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: broken-backend
  namespace: error-lab
spec:
  hosts:
  - broken-backend
  http:
  - route:
    - destination:
        host: broken-backend
        port:
          number: 80

---
# DestinationRule with outlier detection
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: broken-backend
  namespace: error-lab
spec:
  host: broken-backend
  outlierDetection:
    consecutive5xxErrors: 1  # Evict after 1 error
    interval: 10s
    baseEjectionTime: 30s
```

Apply:

```bash
kubectl apply -f services-503-scenario.yaml
kubectl exec -it client -n error-lab -- sh

# Inside client:
curl http://broken-backend
# Returns: HTTP/1.1 503 Service Unavailable
# With body: "upstream connect error or disconnect/reset before headers. reset reason: connection failure"
```

### Section 2.2: Diagnose 503 Errors

**Step 1: Check Endpoints**

```bash
kubectl get endpoints -n error-lab
```

Output:
```
NAME                ENDPOINTS        AGE
broken-backend      <none>           1m
healthy-backend     10.244.0.15:80   1m
```

`broken-backend` has NO endpoints! That's why it returns 503.

**Step 2: Check VirtualService Configuration**

```bash
kubectl describe vs broken-backend -n error-lab
```

Output shows the route destination, but doesn't show why it's failing.

**Step 3: Inspect Envoy Configuration on the Client Pod**

```bash
istioctl proxy-config route client -n error-lab
```

This shows all routes Envoy knows about. Look for your service:

```
NAME                                                  DOMAINS          MATCH                 VIRTUAL SERVICE
broken-backend                                        broken-backend   /*                    broken-backend/error-lab
```

Now check the cluster (upstream) configuration:

```bash
istioctl proxy-config cluster client -n error-lab --fqdn broken-backend
```

Output shows the cluster configuration:

```
NAME             FQDN                                 PORT     TYPE     DIRECTION   AGE
outbound|80||broken-backend.error-lab.svc.cluster.local  broken-backend.error-lab.svc.cluster.local  80   EDS      -          1m
```

Check the endpoints for this cluster:

```bash
istioctl proxy-config endpoint client -n error-lab --cluster "outbound|80||broken-backend.error-lab.svc.cluster.local"
```

Output:
```
ENDPOINT             STATUS      OUTLIER CHECK     IN MESH
<none>               N/A         <none>            N/A
```

NO ENDPOINTS! That's the root cause.

**Step 4: Check Logs on Client Sidecar**

```bash
kubectl logs client -n error-lab -c envoy --tail=50
```

Look for entries like:

```
[2024-01-15T10:30:45.123Z] "GET / HTTP/1.1" 503 UC upstream_connect_error,reset_before_headers
```

`UC` means Upstream Connection error. The breakdown:
- `503` — HTTP 503 response
- `upstream_connect_error` — Couldn't connect to backend
- `reset_before_headers` — Connection was reset before response headers

### Section 2.3: Fix 503 Errors (Multiple Approaches)

**Fix 1: Ensure Endpoints Exist**

```bash
# Check if pods exist for the service
kubectl get pods -n error-lab -l app=broken-backend

# If pods don't exist, create them:
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-backend
  namespace: error-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: broken-backend
  template:
    metadata:
      labels:
        app: broken-backend
    spec:
      containers:
      - name: app
        image: httpbin:latest
        ports:
        - containerPort: 80
EOF

# Wait for pod to start
kubectl wait --for=condition=ready pod -l app=broken-backend -n error-lab --timeout=30s

# Now endpoints should appear
kubectl get endpoints -n error-lab
```

**Fix 2: Adjust Outlier Detection**

If outlier detection is too aggressive, it evicts all healthy endpoints:

```yaml
# destination-rule-less-aggressive.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: broken-backend
  namespace: error-lab
spec:
  host: broken-backend
  outlierDetection:
    consecutive5xxErrors: 5  # Increased from 1
    interval: 30s  # Increased from 10s
    baseEjectionTime: 60s  # Increased from 30s
    maxEjectionPercent: 50  # Never eject more than 50%
    minRequestVolume: 10  # Need at least 10 requests before evicting
```

Apply:

```bash
kubectl apply -f destination-rule-less-aggressive.yaml
kubectl exec -it client -n error-lab -- curl http://broken-backend
# Should succeed now if backend pod is running
```

**Fix 3: Check Circuit Breaker Configuration**

If circuit breaker is limiting connections:

```bash
istioctl proxy-config cluster client -n error-lab -o json | \
  jq '.[] | select(.name | contains("broken-backend")) | .circuitBreakers'
```

Output:
```json
{
  "thresholds": [
    {
      "priority": "DEFAULT",
      "maxConnections": 100,
      "maxPendingRequests": 100,
      "maxRequests": 100,
      "maxRetries": 3
    }
  ]
}
```

If limits are hit, increase them:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: broken-backend
  namespace: error-lab
spec:
  host: broken-backend
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 500  # Increase from default
      http:
        http1MaxPendingRequests: 500
        maxRequestsPerConnection: 2
        h2UpgradePolicy: UPGRADE  # Use HTTP/2
```

### Section 2.4: Troubleshooting 503 Errors Checklist

| Cause | Command to Diagnose | Fix |
|-------|---------------------|-----|
| No endpoints | `kubectl get endpoints <svc>` | Deploy pods matching service selector |
| Outlier detection evicted all pods | `istioctl proxy-config endpoint <pod> --cluster <svc>` | Look for OUTLIER status, adjust DestinationRule |
| Circuit breaker limit hit | `istioctl proxy-config cluster <pod> -o json \| jq '.[] \| .circuitBreakers'` | Increase limits in DestinationRule |
| Wrong destination host | `istioctl proxy-config route <pod>` | Fix VirtualService destination |
| Backend pod crashing | `kubectl logs <backend-pod> -c app --previous` | Fix application |
| Service DNS issue | `kubectl exec <pod> -c envoy -- curl http://<svc>` | Check CoreDNS in kube-system |

---

## Part 3: Connection Resets and Timeouts

### Problem: Connections hang, timeout, or reset before response

Causes:
- Misconfigured retry policies
- Timeout settings too aggressive
- Misconfigured keep-alive
- Network policies blocking traffic
- Sidecar not routing traffic

### Section 3.1: Simulate Connection Timeouts

Create a slow backend service:

```yaml
# timeout-scenario.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: timeout-lab
  labels:
    istio-injection: enabled

---
# Slow backend (5 second response)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-backend
  namespace: timeout-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: slow-backend
  template:
    metadata:
      labels:
        app: slow-backend
    spec:
      containers:
      - name: app
        image: kennethreitz/httpbin:latest
        ports:
        - containerPort: 80
        env:
        - name: PORT
          value: "80"

---
apiVersion: v1
kind: Service
metadata:
  name: slow-backend
  namespace: timeout-lab
spec:
  selector:
    app: slow-backend
  ports:
  - port: 80
    targetPort: 80

---
# VirtualService with aggressive timeout
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: slow-backend
  namespace: timeout-lab
spec:
  hosts:
  - slow-backend
  http:
  - timeout: 1s  # Only 1 second timeout — too short!
    route:
    - destination:
        host: slow-backend
        port:
          number: 80

---
# DestinationRule with retries
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: slow-backend
  namespace: timeout-lab
spec:
  host: slow-backend
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1

---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: timeout-lab
spec:
  containers:
  - name: curl
    image: curlimages/curl
    command: ["sleep", "3600"]
```

Apply:

```bash
kubectl apply -f timeout-scenario.yaml
kubectl exec -it client -n timeout-lab -- sh

# Inside client, try accessing slow endpoint:
curl -v http://slow-backend/delay/3
# After ~1 second, times out
```

### Section 3.2: Diagnose Timeouts

**Step 1: Check VirtualService Timeout**

```bash
kubectl get vs slow-backend -n timeout-lab -o yaml | grep -A 5 timeout:
```

Output:
```yaml
timeout: 1s
```

The timeout is 1 second, but the backend takes 3 seconds. That's the problem.

**Step 2: Inspect Envoy Route Configuration**

```bash
istioctl proxy-config route client -n timeout-lab -o json | \
  jq '.[] | select(.name | contains("slow-backend")) | .virtualHosts[].routes[].typedPerRouteConfig'
```

Look for `timeout` field:

```json
{
  "timeout": "1s"
}
```

**Step 3: Check Sidecar Logs**

```bash
kubectl logs client -n timeout-lab -c envoy --tail=100
```

Look for timeout entries:

```
[2024-01-15T10:30:45.123Z] "GET /delay/3 HTTP/1.1" 504 UC stream_timeout
```

`UC` = Upstream Connection
`stream_timeout` = Request timed out waiting for response

**Step 4: Manually Test Backend Response Time**

```bash
kubectl exec -it slow-backend-<pod-id> -n timeout-lab -- sh

# Inside backend pod, test locally (no Envoy involved):
curl -v http://localhost/delay/3
# Should complete successfully
```

If this works, the backend is fine. The timeout is Envoy-side.

### Section 3.3: Fix Timeouts and Connection Issues

**Fix 1: Increase Timeout**

```bash
kubectl patch vs slow-backend -n timeout-lab --type merge -p \
  '{"spec":{"http":[{"timeout":"10s","route":[{"destination":{"host":"slow-backend"}}]}]}}'

# Test again:
kubectl exec -it client -n timeout-lab -- curl -v http://slow-backend/delay/3
# Should succeed now
```

**Fix 2: Adjust Retry Policy**

Sometimes retries compound the problem. Disable or reduce them:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: slow-backend
  namespace: timeout-lab
spec:
  hosts:
  - slow-backend
  http:
  - timeout: 10s
    retries:
      attempts: 1  # Only 1 attempt, no retries
      perTryTimeout: 5s  # Per-retry timeout
    route:
    - destination:
        host: slow-backend
        port:
          number: 80
```

**Fix 3: Adjust TCP/HTTP Connection Pooling**

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: slow-backend
  namespace: timeout-lab
spec:
  host: slow-backend
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100  # More concurrent connections
      http:
        http1MaxPendingRequests: 100  # More queued requests
        maxRequestsPerConnection: 0  # Unlimited reuse
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
```

**Fix 4: Check Network Policies**

If traffic is actually being blocked by network policies:

```bash
kubectl get networkpolicy -n timeout-lab
kubectl describe networkpolicy <name> -n timeout-lab
```

If a NetworkPolicy is blocking, add an ingress rule:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-slow-backend
  namespace: timeout-lab
spec:
  podSelector:
    matchLabels:
      app: slow-backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          istio-injection: enabled  # Allow from any pod in this namespace
    ports:
    - protocol: TCP
      port: 80
```

### Section 3.4: Timeout and Connection Troubleshooting Checklist

| Symptom | Command | Fix |
|---------|---------|-----|
| Requests timeout | `istioctl proxy-config route <pod> -o json` | Increase `timeout` in VirtualService |
| Connections reset | `kubectl logs <pod> -c envoy \| grep reset` | Check backend logs, increase `maxConnections` |
| Pending requests reject | `istioctl proxy-config cluster <pod> -o json` | Increase `http1MaxPendingRequests` |
| Too many retries | `istioctl proxy-config route <pod>` | Reduce `retries.attempts` or set high `perTryTimeout` |
| Network policies blocking | `kubectl get networkpolicy -A` | Add ingress rule allowing traffic |
| Slow backend | `kubectl exec <pod> -- time curl http://<backend>` | Check backend logs for errors |

---

## Part 4: mTLS Handshake Failures and Policy Conflicts

### Problem: mTLS configuration prevents pod-to-pod communication

When mTLS (mutual TLS) is enforced, both client and server must have proper certificates and policies configured. Mismatches cause connection failures.

### Section 4.1: Understand Istio mTLS

Istio manages certificates automatically via Istiod. For mTLS to work:
1. **Client** must have a valid certificate (injected via sidecar)
2. **Server** must be configured to accept mTLS
3. **PeerAuthentication policy** must allow the connection
4. **AuthorizationPolicy** must permit the traffic

Check current mTLS status:

```bash
# Check PeerAuthentication policies
kubectl get peerauthentication -A

# Check AuthorizationPolicies
kubectl get authorizationpolicy -A

# Check certificates
kubectl get secret -n istio-system | grep istio
```

### Section 4.2: Simulate mTLS Handshake Failure

Create a scenario where mTLS is enforced but not properly configured:

```yaml
# mtls-failure-scenario.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mtls-lab
  labels:
    istio-injection: enabled

---
# Server that doesn't use mTLS (plaintext)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server-plaintext
  namespace: mtls-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: server-plaintext
      version: v1
  template:
    metadata:
      labels:
        app: server-plaintext
        version: v1
    spec:
      containers:
      - name: server
        image: httpbin:latest
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: server-plaintext
  namespace: mtls-lab
spec:
  selector:
    app: server-plaintext
  ports:
  - port: 80
    name: http
    targetPort: 80

---
# Server with mTLS enabled
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server-mtls
  namespace: mtls-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: server-mtls
      version: v1
  template:
    metadata:
      labels:
        app: server-mtls
        version: v1
    spec:
      containers:
      - name: server
        image: httpbin:latest
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: server-mtls
  namespace: mtls-lab
spec:
  selector:
    app: server-mtls
  ports:
  - port: 80
    name: http
    targetPort: 80

---
# Client pod
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: mtls-lab
spec:
  containers:
  - name: curl
    image: curlimages/curl
    command: ["sleep", "3600"]

---
# STRICT mTLS policy for entire namespace
# This enforces mTLS on ALL services
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: mtls-lab
spec:
  mtls:
    mode: STRICT  # Require mTLS from all clients

---
# AuthorizationPolicy: DENY all, then ALLOW specific routes
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: default-deny
  namespace: mtls-lab
spec:
  {}  # Deny all (default action)

---
# Allow only client to server-mtls
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-client-to-server-mtls
  namespace: mtls-lab
spec:
  selector:
    matchLabels:
      app: server-mtls
  rules:
  - from:
    - source:
        principals:
          - "cluster.local/ns/mtls-lab/sa/default"  # client's service account
    to:
    - operation:
        methods: ["GET"]
        ports: ["80"]
```

Apply:

```bash
kubectl apply -f mtls-failure-scenario.yaml
sleep 30  # Wait for all pods to start

# Test client accessing plaintext server (should fail)
kubectl exec -it client -n mtls-lab -- curl -v http://server-plaintext
# Connection refused or "no route to host"

# Test client accessing mTLS server (should fail initially due to policy)
kubectl exec -it client -n mtls-lab -- curl -v http://server-mtls
# Also fails
```

### Section 4.3: Diagnose mTLS Handshake Failures

**Step 1: Check PeerAuthentication Policies**

```bash
kubectl get peerauthentication -n mtls-lab
kubectl describe peerauthentication default -n mtls-lab
```

Output:
```
spec:
  mtls:
    mode: STRICT  # This requires mTLS from all clients
```

**Step 2: Check AuthorizationPolicy**

```bash
kubectl get authorizationpolicy -n mtls-lab
kubectl describe authorizationpolicy allow-client-to-server-mtls -n mtls-lab
```

**Step 3: Inspect Envoy Listener Configuration**

See how Envoy is configured to handle mTLS:

```bash
istioctl proxy-config listener client -n mtls-lab -o json | \
  jq '.[] | select(.name | contains("server")) | .filterChains'
```

Look for `transportSocket` with `tlsContext`:

```json
{
  "transportSocket": {
    "name": "tls",
    "typedConfig": {
      "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext",
      "commonTlsContext": {
        "tlsCertificates": [...]  # Client certificate
      }
    }
  }
}
```

If `transportSocket` is missing, mTLS is NOT configured.

**Step 4: Check Sidecar Logs for TLS Errors**

```bash
kubectl logs client -n mtls-lab -c envoy | grep -i "tls\|ssl\|certificate\|handshake"
```

Look for errors like:

```
TLS certificate chain validation failed
TLS handshake failed
SSL_ERROR_RX_RECORD_OVERFLOW
```

**Step 5: Check Server's mTLS Certificate**

Verify the server has a valid certificate:

```bash
kubectl get secret -n mtls-lab | grep tls
# Should show certificates issued by Istio

# Inspect certificate details:
kubectl get secret istio.io/ds/mtls-lab-server-mtls -n mtls-lab -o json | \
  jq -r '.data."tls.crt" | @base64d' | openssl x509 -noout -text
```

**Step 6: Use istioctl to Check Certificate Status**

```bash
istioctl authn tls-check client -n mtls-lab
```

Output shows TLS configuration:

```
HOST:PORT           CURL  TLS-MODE     DESTINATION RULE   AUTHN POLICY     DESTINATION     UP-TO-DATE
server-mtls:80      -     STRICT mTLS  -                  default          -               true
```

`CURL: -` means curl can't reach it (likely TLS mismatch).

### Section 4.4: Fix mTLS Issues

**Fix 1: Allow Plaintext Traffic**

If server doesn't support mTLS, allow plaintext:

```bash
kubectl patch peerauthentication default -n mtls-lab --type merge -p \
  '{"spec":{"mtls":{"mode":"PERMISSIVE"}}}'

# Test again:
kubectl exec -it client -n mtls-lab -- curl -v http://server-plaintext
# Should work now
```

**Fix 2: Ensure AuthorizationPolicy Permits Traffic**

```bash
# Update AuthorizationPolicy to allow traffic
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-all-in-mtls-lab
  namespace: mtls-lab
spec:
  rules:
  - {}  # Allow all
EOF

# Test:
kubectl exec -it client -n mtls-lab -- curl -v http://server-mtls
# Should work now
```

**Fix 3: Verify Certificate Chain**

```bash
# Check if Istio issued certificates correctly
kubectl get secret -n mtls-lab -o wide | grep istio

# Verify certificate is not expired:
kubectl get secret istio.io/ds/mtls-lab-client -n mtls-lab -o json | \
  jq -r '.data."tls.crt" | @base64d' | openssl x509 -noout -dates
```

If certificate is expired, Istiod will issue a new one automatically.

**Fix 4: Check Istiod is Running and Healthy**

```bash
kubectl get pods -n istio-system | grep istiod
kubectl logs -n istio-system -l app=istiod | grep -i "certificate\|tls\|error"

# If unhealthy, restart:
kubectl rollout restart deployment istiod -n istio-system
```

### Section 4.5: Policy Conflict Resolution

When multiple policies conflict, diagnose systematically:

**Step 1: List All Policies Affecting a Pod**

```bash
# Check namespace-level policies
kubectl get peerauthentication -n mtls-lab
kubectl get authorizationpolicy -n mtls-lab

# Check cluster-level policies (root namespace)
kubectl get peerauthentication -n istio-system
kubectl get authorizationpolicy -n istio-system
```

**Step 2: Identify Which Policy Applies**

Kubernetes applies policies hierarchically:
1. **Namespace-level policies** override cluster-level
2. **More specific selectors** override broader ones
3. **Last applied** wins on ties

Example:

```yaml
# Cluster-wide: deny all
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: istio-system  # Cluster-wide
spec:
  rules: []  # Deny all

---
# Namespace override: allow all
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-all
  namespace: mtls-lab  # Namespace-specific (wins)
spec:
  rules:
  - {}  # Allow all
```

**Step 3: Test Policy Impact**

```bash
# Simulate the policy by checking if traffic would be allowed:
kubectl auth can-i --list --as=system:serviceaccount:mtls-lab:default \
  -n mtls-lab

# Or enable audit logging:
kubectl patch authorizationpolicy allow-all -n mtls-lab --type merge -p \
  '{"spec":{"auditLogging":{"enable":true}}}'

# Check Envoy logs:
kubectl logs client -n mtls-lab -c envoy | grep -i "denied\|audit"
```

### Section 4.6: mTLS and Policy Troubleshooting Checklist

| Issue | Command | Fix |
|-------|---------|-----|
| TLS handshake fails | `istioctl authn tls-check <pod>` | Check PeerAuthentication mode (STRICT vs PERMISSIVE) |
| Certificate expired | `kubectl get secret <cert> -o json \| jq '.data."tls.crt"' \| openssl x509 -dates` | Wait for Istiod to issue new cert, or restart Istiod |
| Policy denies traffic | `kubectl get authorizationpolicy -A` | Update policy rules or check selector mismatch |
| mTLS not enforced | `istioctl proxy-config listener <pod> -o json \| jq '.[] \| .filterChains'` | Ensure PeerAuthentication set to STRICT |
| Certificate chain invalid | `kubectl logs -n istio-system -l app=istiod \| grep certificate` | Check Istiod logs, restart if needed |
| Multiple policies conflict | Check namespace and cluster-level policies | Namespace-specific policies override cluster-wide |

---

## Part 5: Advanced Envoy Inspection

### Using istioctl to Inspect Envoy Configuration

The most powerful tool for Istio troubleshooting is `istioctl proxy-config`. It shows exactly what Envoy is configured to do.

### Section 5.1: Proxy Configuration Commands

```bash
# Show all routes
istioctl proxy-config route <pod> [-n namespace] [-o wide]

# Show clusters (upstream services)
istioctl proxy-config cluster <pod> [-n namespace]

# Show listeners (inbound/outbound ports)
istioctl proxy-config listener <pod> [-n namespace]

# Show endpoints (actual backend IPs)
istioctl proxy-config endpoint <pod> [-n namespace] [--cluster <name>]

# Show bootstrap configuration
istioctl proxy-config bootstrap <pod> [-n namespace]

# Show Envoy's entire configuration in JSON
istioctl proxy-config all <pod> [-n namespace] -o json | jq . | less
```

### Section 5.2: Practical Debugging Examples

**Example 1: Trace a Request's Path**

When a client pod tries to reach a service:

```bash
# 1. List all routes the client knows about
istioctl proxy-config route client-pod -n default

# 2. Find the specific route for your service
istioctl proxy-config route client-pod -n default | grep "my-service"

# 3. Check what cluster (upstream) that route points to
istioctl proxy-config route client-pod -n default --name 9080 -o json | \
  jq '.[] | select(.virtualHosts[].routes[].match.prefix=="/") | .virtualHosts[].routes[].route.cluster'

# 4. Check endpoints for that cluster
istioctl proxy-config endpoint client-pod -n default \
  --cluster "outbound|80||my-service.default.svc.cluster.local"

# 5. Each endpoint is a backend pod. If list is empty, there's no endpoint!
```

**Example 2: Inspect Circuit Breaker Settings**

```bash
istioctl proxy-config cluster client-pod -n default -o json | \
  jq '.[] | select(.name | contains("my-service")) | .circuitBreakers'
```

Output shows current limits. If requests are being rejected, limits might be too low.

**Example 3: Check TLS Configuration**

```bash
# Show listeners (inbound/outbound) with TLS
istioctl proxy-config listener client-pod -n default -o json | \
  jq '.[] | select(.filterChains[].transportSocket) | {name, filterChains}'

# Check if server expects mTLS:
istioctl proxy-config listener server-pod -n default | grep -i tls
```

**Example 4: Verify VirtualService Implementation**

```bash
# VirtualService in YAML
kubectl get vs my-service -n default -o yaml

# How it's actually configured in Envoy:
istioctl proxy-config route client-pod -n default -o json | \
  jq '.[] | select(.name | contains("my-service"))'
```

Compare the YAML spec with Envoy config to see if it's properly applied.

### Section 5.3: Proxy Status and Sync Issues

Check if all proxies are properly synced with Istiod:

```bash
istioctl proxy-status
```

Output:
```
NAME                                               CDS  LDS  RDS  EDS  PILOT AGEN VERSION
test-client-7d6d4f8d9f-abc12                       OK   OK   OK   OK   14s   13s  1.17.0
web-deployment-6b8f7c9d8-xyz98                     OK   OK   OK   OK   15s   12s  1.17.0
upstream-service-5a3e2b1d0-def45                   SYNCED OK   SYNCED OK   16s   11s  1.17.0
```

States:
- `OK` / `SYNCED` — Proxy is in sync with Istiod
- `STALE` — Proxy hasn't been updated recently
- `NOT SENT` — Istiod hasn't sent config yet

If a proxy is `STALE`, check:

```bash
# Check Istiod logs
kubectl logs -n istio-system -l app=istiod | tail -50

# Check pod's sidecar logs
kubectl logs <pod> -c envoy | tail -50

# Restart Istiod if stuck
kubectl rollout restart deployment istiod -n istio-system
```

### Section 5.4: JSON Inspection Deep Dive

For complex debugging, export full configuration:

```bash
# Save entire Envoy config to file
istioctl proxy-config all <pod> -n <namespace> -o json > envoy-config.json

# Analyze specific components
cat envoy-config.json | jq '.listeners'      # Inbound/outbound listeners
cat envoy-config.json | jq '.clusters'       # Upstream service clusters
cat envoy-config.json | jq '.routeConfigs'   # Route configurations
cat envoy-config.json | jq '.endpoints'      # Backend endpoints
```

Common queries:

```bash
# Find all routes
jq '.routeConfigs[].virtualHosts[].routes' envoy-config.json

# Find timeout configuration
jq '.routeConfigs[].virtualHosts[].routes[] | select(.timeout) | {timeout}' envoy-config.json

# Find circuit breaker config
jq '.clusters[] | select(.circuitBreakers) | {name, circuitBreakers}' envoy-config.json

# Check for mTLS (look for tlsContext)
jq '.clusters[] | select(.transportSocket) | {name}' envoy-config.json
```

---

## Part 6: Complete Lab Scenario - Multi-Issue Istio Debugging

Combine all issues: sidecar injection, 503 errors, timeout, mTLS failure, and policy conflicts.

### Section 6.1: The Broken Microservices Stack

```yaml
# complete-istio-scenario.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: complete-lab
  labels:
    istio-injection: enabled

---
# Frontend service (has sidecar)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: complete-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: complete-lab
spec:
  selector:
    app: frontend
  ports:
  - port: 80

---
# Backend service (no sidecar — not labeled!)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: complete-lab
  labels:
    sidecar: "false"  # Hint: don't inject
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: app
        image: httpbin:latest
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: complete-lab
spec:
  selector:
    app: backend
  ports:
  - port: 80

---
# VirtualService with aggressive timeout
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: backend
  namespace: complete-lab
spec:
  hosts:
  - backend
  http:
  - timeout: 500ms  # Too short for httpbin
    route:
    - destination:
        host: backend

---
# DestinationRule with tight circuit breaker
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: backend
  namespace: complete-lab
spec:
  host: backend
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 2  # Too low

---
# Strict mTLS policy (conflicts with missing sidecar on backend)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: complete-lab
spec:
  mtls:
    mode: STRICT

---
# Deny-all AuthorizationPolicy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: default-deny
  namespace: complete-lab
spec:
  {}

---
# Client pod (has sidecar)
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: complete-lab
spec:
  containers:
  - name: curl
    image: curlimages/curl
    command: ["sleep", "3600"]
```

Apply:

```bash
kubectl apply -f complete-istio-scenario.yaml
sleep 20

# Try to access backend from frontend
kubectl exec -it client -n complete-lab -- curl -v http://backend
# Multiple failures:
# 1. No route (sidecar not injected on backend)
# 2. Or 503 if injected but authorization denied
# 3. Or timeout if request takes >500ms
```

### Section 6.2: Systematic Diagnosis

Run through all diagnostics:

```bash
# 1. Check sidecar injection
kubectl get pods -n complete-lab -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
# frontend: app envoy
# backend: app (NO envoy!)
# client: curl envoy

# 2. Check service endpoints
kubectl get endpoints -n complete-lab

# 3. Check VirtualService
kubectl get vs -n complete-lab
kubectl describe vs backend -n complete-lab

# 4. Check policies
kubectl get peerauthentication -n complete-lab
kubectl get authorizationpolicy -n complete-lab

# 5. Inspect Envoy route on client
istioctl proxy-config route client -n complete-lab

# 6. Check endpoints available for backend cluster
istioctl proxy-config endpoint client -n complete-lab \
  --cluster "outbound|80||backend.complete-lab.svc.cluster.local"

# 7. Check Envoy logs for errors
kubectl logs client -n complete-lab -c envoy | grep -i "denied\|timeout\|tls"
```

### Section 6.3: Fix All Issues in Order

**Fix 1: Enable Sidecar Injection on Backend**

```bash
# Add sidecars to backend deployment
kubectl patch deployment backend -n complete-lab --type json -p \
  '[{"op":"add","path":"/spec/template/metadata/labels/injected","value":"true"}]'

# Restart to trigger injection
kubectl rollout restart deployment backend -n complete-lab

# Verify sidecar injected
kubectl get pods -n complete-lab -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
# backend: app envoy (now has sidecar!)
```

**Fix 2: Increase Timeout**

```bash
kubectl patch vs backend -n complete-lab --type merge -p \
  '{"spec":{"http":[{"timeout":"5s","route":[{"destination":{"host":"backend"}}]}]}}'
```

**Fix 3: Increase Circuit Breaker Limits**

```bash
kubectl patch dr backend -n complete-lab --type merge -p \
  '{"spec":{"trafficPolicy":{"connectionPool":{"http":{"http1MaxPendingRequests":100}}}}}'
```

**Fix 4: Allow Traffic in AuthorizationPolicy**

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-client-to-backend
  namespace: complete-lab
spec:
  selector:
    matchLabels:
      app: backend
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/complete-lab/sa/default"
    to:
    - operation:
        methods: ["GET"]
        ports: ["80"]
EOF
```

**Fix 5: Test End-to-End**

```bash
kubectl exec -it client -n complete-lab -- curl -v http://backend
# Should succeed now and return httpbin welcome page
```

---

## Part 7: Monitoring and Observability

### Kiali Dashboard (Visual Troubleshooting)

Kiali provides a graphical view of service mesh traffic and issues:

```bash
# Access Kiali (if installed)
kubectl port-forward -n istio-system svc/kiali 20000:20000
# Open http://localhost:20000 in browser
```

In Kiali, you can:
- See service topology and traffic flow
- Identify pods with high error rates (red edges)
- View details of service interactions
- Check deployment status

### Prometheus Metrics

Query metrics related to your debugging:

```bash
# Port-forward to Prometheus
kubectl port-forward -n istio-system svc/prometheus 9090:9090

# Query examples in browser (http://localhost:9090):

# 4xx errors
rate(istio_request_total{response_code=~"4.."}[5m])

# 5xx errors
rate(istio_request_total{response_code=~"5.."}[5m])

# Request latency
histogram_quantile(0.95, rate(istio_request_duration_milliseconds_bucket[5m]))

# Active connections
envoy_upstream_cx{}

# Circuit breaker rejections
envoy_cluster_circuit_breakers_default_cx_open{}
```

### Jaeger Distributed Tracing

View request traces across services:

```bash
# Port-forward to Jaeger
kubectl port-forward -n istio-system svc/jaeger-collector 16686:16686

# Open http://localhost:16686 in browser
```

Traces show:
- Which services were called
- Response times at each step
- Errors and their location
- Network latency

---

## Part 8: Cleanup

```bash
# Remove all lab namespaces
kubectl delete namespace inject-lab error-lab timeout-lab mtls-lab complete-lab

# Or keep bookinfo for learning:
# kubectl delete namespace bookinfo
```

---

## Quick Reference: Istio Debugging Flowchart

```
Istio Traffic Issue?

├─ Pods don't connect at all
│  ├─ Sidecar injected?
│  │  └─ Run: kubectl get pods <pod> -o yaml | grep -c envoy
│  │     └─ Fix: kubectl label ns <ns> istio-injection=enabled
│  │
│  ├─ No endpoints for service?
│  │  └─ Run: kubectl get endpoints <svc>
│  │     └─ Fix: Ensure pods match service selector
│  │
│  └─ mTLS handshake failed?
│     └─ Run: istioctl authn tls-check <pod>
│        └─ Fix: Check PeerAuthentication mode, certificates
│
├─ Connections timeout or reset
│  ├─ Timeout too aggressive?
│  │  └─ Run: istioctl proxy-config route <pod> -o json | jq '.[] | select(.timeout)'
│  │     └─ Fix: Increase timeout in VirtualService
│  │
│  ├─ Circuit breaker limit hit?
│  │  └─ Run: istioctl proxy-config cluster <pod> -o json | jq '.[] | .circuitBreakers'
│  │     └─ Fix: Increase limits in DestinationRule
│  │
│  └─ Network policy blocking?
│     └─ Run: kubectl get networkpolicy -A
│        └─ Fix: Add ingress rule
│
├─ 503 Service Unavailable
│  ├─ No healthy endpoints?
│  │  └─ Run: istioctl proxy-config endpoint <pod> --cluster <svc>
│  │     └─ Fix: Deploy pods, check health
│  │
│  ├─ Outlier detection evicted all pods?
│  │  └─ Run: kubectl describe dr <name>
│  │     └─ Fix: Relax outlier detection settings
│  │
│  └─ Wrong destination host?
│     └─ Run: istioctl proxy-config route <pod>
│        └─ Fix: Check VirtualService destination
│
├─ Authorization denied (403)
│  ├─ Policy denies traffic?
│  │  └─ Run: kubectl get authorizationpolicy -A
│  │     └─ Fix: Update rules to allow traffic
│  │
│  └─ mTLS policy conflicts?
│     └─ Run: kubectl get peerauthentication -A
│        └─ Fix: Set mode to PERMISSIVE or fix mTLS setup
│
└─ Slow or high latency
   ├─ High error rate?
   │  └─ Run: istioctl proxy-config all <pod> -o json | jq '.endpoints' | grep -i outlier
   │     └─ Fix: Check backend health, reduce outlier detection
   │
   ├─ Retries adding latency?
   │  └─ Run: istioctl proxy-config route <pod> -o json | jq '.[] | .retries'
   │     └─ Fix: Reduce retry attempts or increase perTryTimeout
   │
   └─ Slow backend?
      └─ Run: kubectl logs <backend-pod> -c app | tail -50
         └─ Fix: Profile and optimize backend
```

---

## Essential istioctl Commands Cheat Sheet

```bash
# Version and status
istioctl version                                 # Istio version
istioctl proxy-status                            # All proxies sync status

# Configuration inspection
istioctl proxy-config route <pod> [-n ns]        # Routes (VirtualService impl)
istioctl proxy-config cluster <pod> [-n ns]      # Clusters (upstreams)
istioctl proxy-config listener <pod> [-n ns]     # Listeners (ports)
istioctl proxy-config endpoint <pod> [-n ns]     # Endpoints (backend IPs)
istioctl proxy-config bootstrap <pod> [-n ns]    # Bootstrap config

# JSON output for analysis
istioctl proxy-config all <pod> -o json | jq .  # Full Envoy config

# Authentication and policy
istioctl authn tls-check <pod> [-n ns]           # TLS config check
istioctl analyze [-n ns]                         # Validate configs for issues

# Logs and debug
kubectl logs <pod> -c envoy [-n ns] [-f]         # Sidecar logs
kubectl logs <pod> -c envoy [-n ns] --tail=100   # Last 100 lines

# Traffic debugging
kubectl exec <pod> [-n ns] -- curl http://<svc>  # Test connectivity
kubectl exec <pod> [-n ns] -it -- /bin/sh        # Shell access

# Port forwarding
kubectl port-forward -n istio-system \
  svc/kiali 20000:20000                          # Kiali dashboard
kubectl port-forward -n istio-system \
  svc/prometheus 9090:9090                       # Prometheus metrics
kubectl port-forward -n istio-system \
  svc/jaeger-collector 16686:16686               # Jaeger traces
```

---

## Common Istio Error Messages and Fixes

| Error Message | Cause | Fix |
|---------------|-------|-----|
| `upstream connect error or disconnect/reset before headers` | No healthy endpoints or connection refused | Check endpoints with `kubectl get endpoints` |
| `TLS certificate chain validation failed` | mTLS certificate issue | Check `istioctl authn tls-check`, restart Istiod |
| `HTTP/1.1 503 Service Unavailable` | Service has no endpoints or all outlier-detected | Check endpoints, adjust outlier detection |
| `stream timeout` | Timeout too short for backend | Increase `timeout` in VirtualService |
| `RBAC: access denied` | AuthorizationPolicy denies traffic | Check policy rules with `kubectl describe authorizationpolicy` |
| `connection reset by peer` | Sidecar not properly injected or network issue | Check sidecar with `kubectl get pod -o yaml \| grep envoy` |
| `SSL_ERROR_RX_RECORD_OVERFLOW` | mTLS protocol mismatch | Ensure both sides use mTLS (check PeerAuthentication) |
| `Upstream cluster not found` | VirtualService references non-existent cluster | Verify cluster exists in DestinationRule or service selector |

---

## Learning Outcomes

After completing this lab, you should be able to:

- [ ] Understand Istio's sidecar injection mechanism and troubleshoot injection failures
- [ ] Diagnose and fix 503 errors by inspecting endpoints and outlier detection
- [ ] Identify timeout and retry issues using VirtualService and DestinationRule
- [ ] Debug connection resets and network problems
- [ ] Understand mTLS handshake failures and certificate issues
- [ ] Use AuthorizationPolicy and PeerAuthentication to secure traffic
- [ ] Inspect Envoy configuration with `istioctl proxy-config` commands
- [ ] Read Envoy logs to understand traffic routing decisions
- [ ] Use Kiali, Prometheus, and Jaeger for observability
- [ ] Systematically diagnose multi-issue scenarios
- [ ] Understand the relationship between Kubernetes objects and Envoy config

---

## Additional Resources

- [Istio Troubleshooting Guide](https://istio.io/latest/docs/ops/troubleshooting/)
- [istioctl proxy-config Documentation](https://istio.io/latest/docs/reference/commands/istioctl/#proxy-config)
- [Envoy Configuration Reference](https://www.envoyproxy.io/docs/envoy/latest/)
- [Istio Security Policies](https://istio.io/latest/docs/concepts/security/)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Kiali Documentation](https://kiali.io/docs/)
- [Bookinfo Sample Application](https://istio.io/latest/docs/examples/bookinfo/)

---

## Appendix: Quick Start — Deploy and Test Everything

Run this one command to set up all 6 lab scenarios:

```bash
# 1. Ensure Istio installed with sidecar injection enabled
istioctl install --set profile=demo -y

# 2. Enable sidecar injection on default namespace
kubectl label namespace default istio-injection=enabled

# 3. Deploy BookInfo sample
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.17/samples/bookinfo/platform/kube/bookinfo.yaml

# 4. Create all lab scenarios (from above YAMLs)
kubectl apply -f crash-loop-deployment.yaml
kubectl apply -f services-503-scenario.yaml
kubectl apply -f timeout-scenario.yaml
kubectl apply -f mtls-failure-scenario.yaml
kubectl apply -f complete-istio-scenario.yaml

# 5. Start learning!
# Open three terminals:
# Terminal 1: kubectl port-forward -n istio-system svc/kiali 20000:20000
# Terminal 2: Watch events: kubectl get events -A --watch
# Terminal 3: Run diagnostics and fixes from the lab

# 6. Clean up when done
kubectl delete namespace inject-lab error-lab timeout-lab mtls-lab complete-lab
istioctl uninstall --purge
```

Enjoy your Istio troubleshooting journey!

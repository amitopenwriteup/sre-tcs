# Istio Service Mesh Workshop

## Module 1: Why a Service Mesh? The Problems Istio Solves

### The Microservices Networking Problem

As applications move from monoliths to microservices, service-to-service communication becomes a first-class engineering problem. Every service ends up needing the same cross-cutting capabilities:

- Service discovery and load balancing
- Retries, timeouts, and circuit breaking
- Encryption between services
- Authentication and authorization
- Observability (metrics, logs, traces)
- Traffic control (canary releases, A/B testing, fault injection)

**Without a service mesh**, teams solve these problems in one of two ways, both painful:

1. **Baked into application code** — every service reimplements retry logic, TLS, auth headers, etc. This is duplicated effort across every language/team, hard to keep consistent, and couples business logic to networking concerns.
2. **Shared libraries** — a common library handles this instead of duplicating code. Better, but now every service must be on the same language/runtime, and upgrading the library means redeploying *every* service.

### What Istio Solves

Istio moves all of this networking logic **out of the application** and into the infrastructure layer, transparently, without code changes.

| Problem | How Istio Solves It |
|---|---|
| **Traffic management** | Fine-grained routing, retries, timeouts, circuit breaking, traffic splitting — configured declaratively, not in code |
| **Security** | Automatic mutual TLS (mTLS) between services, strong workload identity, fine-grained access policies |
| **Observability** | Automatic metrics, distributed tracing, and access logs for every service call — with zero app instrumentation |
| **Policy enforcement** | Rate limiting, quota management, and access control applied consistently across the mesh |
| **Resilience** | Automatic retries, timeouts, circuit breakers, and outlier detection without app-level code |
| **Language/framework independence** | Since the logic lives in a sidecar proxy, it works identically whether the service is written in Go, Java, Python, Node, etc. |

### The Core Idea

> Instead of embedding networking logic in every service, place a lightweight proxy next to every service instance. Let the proxy handle communication, and centrally control all the proxies.

This is the essence of a service mesh: **the network becomes programmable, observable, and secure by default — application developers stop thinking about it.**

---

## Module 2: Istio Architecture

Istio has two logical planes: the **control plane** and the **data plane**.

```
┌─────────────────────────────────────────────────────┐
│                    CONTROL PLANE                     │
│                       (istiod)                       │
│                                                        │
│   Pilot          Citadel          Galley               │
│  (config &       (certs &        (config              │
│   discovery)      identity)       validation)          │
└───────────────────────┬───────────────────────────────┘
                         │  (config, certs, policy pushed
                         │   down via xDS APIs)
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│   Pod A        │ │   Pod B        │ │   Pod C        │
│ ┌───────────┐  │ │ ┌───────────┐  │ │ ┌───────────┐  │
│ │   App      │  │ │ │   App      │  │ │ │   App      │  │
│ │ container  │  │ │ │ container  │  │ │ │ container  │  │
│ └─────┬─────┘  │ │ └─────┬─────┘  │ │ └─────┬─────┘  │
│       │ localhost      │ localhost      │ localhost │
│ ┌─────▼─────┐  │ │ ┌─────▼─────┐  │ │ ┌─────▼─────┐  │
│ │  Envoy     │◄─┼─┼─┤  Envoy     │◄─┼─┼─┤  Envoy     │  │
│ │  sidecar   │  │ │ │  sidecar   │  │ │ │  sidecar   │  │
│ └───────────┘  │ │ └───────────┘  │ │ └───────────┘  │
└───────────────┘ └───────────────┘ └───────────────┘
                DATA PLANE (mTLS between sidecars)
```

### Control Plane: `istiod`

Since Istio 1.5, the control plane components were consolidated into a single binary/deployment called **istiod**. Conceptually it still performs three distinct jobs:

- **Pilot (traffic management / config distribution)**
  Converts high-level Istio configuration (VirtualServices, DestinationRules, Gateways) into Envoy-specific configuration, and pushes it to every sidecar using Envoy's xDS APIs (LDS, RDS, CDS, EDS). Watches the Kubernetes API for services/endpoints and keeps proxies updated in real time as pods scale up/down.

- **Citadel (security / identity)**
  Acts as the mesh's certificate authority (CA). Issues and rotates short-lived X.509 certificates to every workload, giving each service a cryptographic identity (SPIFFE-based). This identity is what makes automatic mutual TLS possible — proxies use these certs to authenticate each other.

- **Galley (configuration ingestion & validation)**
  Validates user-authored Istio configuration (YAML) before it's used, and abstracts away the underlying platform (Kubernetes, VMs) so Pilot doesn't need platform-specific logic.

**Key point:** istiod does *not* sit in the data path. If istiod goes down, existing sidecars keep working with their last-known configuration — only new config changes and new workloads are affected.

### Data Plane: Envoy Sidecars

The data plane is made of **Envoy proxy** instances, one deployed as a **sidecar container** alongside every application container in a pod.

What the sidecar actually does:

- **Intercepts all inbound/outbound traffic** for the pod via `iptables` rules (or the newer ambient mode's alternatives), so the app doesn't need to know a proxy exists.
- **Load balances** requests to healthy endpoints (round robin, least-request, etc.).
- **Terminates and originates mTLS** — encrypts traffic leaving the pod, decrypts traffic entering it, using the identity/certs issued by istiod.
- **Applies traffic rules** — timeouts, retries, circuit breaking, fault injection, traffic splitting for canary rollouts.
- **Collects telemetry** — request-level metrics, distributed tracing spans, and access logs, automatically, for every hop.
- **Enforces policy** — authorization decisions (who can talk to whom), rate limits.

Because the sidecar handles all of this, the **application code has zero awareness** of Istio. A service just calls `http://other-service:8080` like normal, and the sidecar quietly does the interception, encryption, routing, and telemetry.

### Why Two Planes?

Separating control from data is what makes Istio scale and stay resilient:

- **Data plane** must be fast and always up — it's in the request path for every single call. Envoy is written in C++ for exactly this reason.
- **Control plane** can be slower and eventually-consistent — it's only pushing *configuration*, not proxying live traffic. If it restarts or lags, the mesh keeps functioning on cached config.

---

## Module 3: Installation and Sidecar Injection

### Installation Options

Istio can be installed in a few ways, in increasing order of production-readiness:

1. **`istioctl install`** — the recommended CLI-based approach using built-in configuration profiles
2. **Istio Operator** — a Kubernetes operator that manages the Istio lifecycle declaratively
3. **Helm charts** — for teams standardized on Helm-based GitOps workflows

#### Quick Install with `istioctl`

```bash
# Download istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install using the "demo" profile (good for workshops/learning)
istioctl install --set profile=demo -y

# Verify the control plane is running
kubectl get pods -n istio-system
```

Common installation profiles:

| Profile | Use Case |
|---|---|
| `default` | Production-oriented, sensible defaults |
| `demo` | All features enabled, for learning/demos (not for prod) |
| `minimal` | Only istiod, no extra components — build up manually |
| `remote` | For multi-cluster secondary clusters |
| `empty` | Installs nothing — for fully custom configs |

#### Verifying the Install

```bash
istioctl verify-install
kubectl get svc -n istio-system
```

You should see `istiod` running, plus (depending on profile) ingress/egress gateway services.

### Sidecar Injection

Sidecar injection is the mechanism that actually adds the Envoy proxy container into your application pods. There are two ways to trigger it.

#### 1. Automatic Injection (recommended)

Label the **namespace**, and Istio's mutating webhook automatically injects a sidecar into every new pod created in that namespace:

```bash
kubectl label namespace default istio-injection=enabled
```

After labeling, **any pod deployed afterward** picks up the sidecar automatically — no changes needed to individual Deployment manifests. Existing pods must be restarted to pick it up:

```bash
kubectl rollout restart deployment -n default
```

Check it worked — pods should show `2/2` containers ready (app + `istio-proxy`):

```bash
kubectl get pods -n default
NAME                        READY   STATUS
reviews-v1-7f4bb9b9-abcde   2/2     Running
```

#### 2. Manual Injection

Useful for one-off testing, or namespaces where you don't want *all* pods injected:

```bash
istioctl kube-inject -f app.yaml | kubectl apply -f -
```

This rewrites the pod spec on the fly, adding the `istio-proxy` container and init container before applying it.

#### How Injection Actually Works Under the Hood

1. **Init container (`istio-init`)** runs first and sets up `iptables` rules inside the pod's network namespace, redirecting all inbound/outbound traffic through the sidecar's ports.
2. **`istio-proxy` container** starts alongside the app container — this *is* the Envoy sidecar.
3. On startup, the sidecar fetches its initial configuration and certificates from **istiod**, then continues to receive live updates via the xDS streaming protocol.

```
Pod Spec (before injection)          Pod Spec (after injection)
┌─────────────────┐                  ┌─────────────────────────┐
│  app container    │      ──►        │  init container         │
└─────────────────┘                  │  (istio-init, sets       │
                                      │   iptables rules)        │
                                      ├─────────────────────────┤
                                      │  app container            │
                                      ├─────────────────────────┤
                                      │  istio-proxy container   │
                                      │  (Envoy sidecar)         │
                                      └─────────────────────────┘
```

#### Disabling Injection for a Specific Pod

Even in an injection-enabled namespace, you can opt a single pod out:

```yaml
metadata:
  annotations:
    sidecar.istio.io/inject: "false"
```

### Workshop Exercise

1. Install Istio using the `demo` profile.
2. Label the `default` namespace for auto-injection.
3. Deploy the [Istio Bookinfo sample app](https://istio.io/latest/docs/examples/bookinfo/) and confirm each pod shows `2/2` containers.
4. Run `istioctl proxy-status` to confirm all sidecars are synced with istiod.
5. Run `istioctl proxy-config cluster <pod-name>` to inspect what Envoy config was actually pushed down.

---

## Key Takeaways

- Istio solves cross-cutting networking concerns (traffic control, security, observability) **without** touching application code.
- The **control plane (istiod)** owns configuration, service discovery, and certificate issuance — it stays out of the live request path.
- The **data plane (Envoy sidecars)** does the actual work: routing, load balancing, mTLS, telemetry — for every single request.
- **Sidecar injection** (automatic via namespace label, or manual via `istioctl kube-inject`) is what wires a pod into the mesh.

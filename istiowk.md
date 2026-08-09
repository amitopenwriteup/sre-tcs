# Lab: Istio Installation & Sidecar Injection

**Goal:** Install Istio, deploy a sample app, and verify sidecar injection end-to-end.

**Prerequisites:** A running Kubernetes cluster (minikube, kind, or a cloud cluster) with `kubectl` configured, and cluster-admin access.

---

## Step 1: Download and Install `istioctl`

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

istioctl version --remote=false
```

### Troubleshooting: `cd: istio-*.tar.gz: Not a directory`

If you hit this error, the script downloaded the tarball but didn't extract it, so `istio-*` matched the `.tar.gz` file itself rather than a folder.

**1. Check what you have:**

```bash
ls -la istio-*.tar.gz
```

**2. Confirm the file isn't a truncated/corrupted download:**

```bash
file istio-*.tar.gz
```

- Expected: `gzip compressed data`
- If instead you see `ASCII text` or `HTML document`, the download failed and grabbed an error page instead of the real archive.

**3. Extract manually (or re-download first if corrupted):**

```bash
# If corrupted, re-download directly first:
curl -LO https://github.com/istio/istio/releases/download/1.30.3/istio-1.30.3-linux-amd64.tar.gz

tar -xzf istio-1.30.3-linux-amd64.tar.gz
cd istio-1.30.3
export PATH=$PWD/bin:$PATH
istioctl version --remote=false
```

Once this prints a client version, continue to Step 2.

---

## Step 2: Install Istio (demo profile)

```bash
istioctl install --set profile=demo -y
```

Verify:

```bash
kubectl get pods -n istio-system
istioctl verify-install
```

You should see `istiod`, `istio-ingressgateway`, and `istio-egressgateway` pods `Running`.

---

## Step 3: Enable Automatic Sidecar Injection

```bash
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled

kubectl get namespace bookinfo --show-labels
```

---

## Step 4: Deploy the Bookinfo Sample App

```bash
kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml
```

Confirm each pod has **2/2** containers ready (app + `istio-proxy`):

```bash
kubectl get pods -n bookinfo

NAME                              READY   STATUS
details-v1-...                    2/2     Running
productpage-v1-...                 2/2     Running
ratings-v1-...                     2/2     Running
reviews-v1-...                     2/2     Running
reviews-v2-...                     2/2     Running
reviews-v3-...                     2/2     Running
```

If a pod shows `1/1`, it was created *before* the namespace label was applied — roll it out again:

```bash
kubectl rollout restart deployment -n bookinfo
```

---

## Step 5: Manual Injection (compare against automatic)

Create a plain namespace with no label, and inject a pod manually instead:

```bash
kubectl create namespace manual-test
istioctl kube-inject -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml \
  -n manual-test | kubectl apply -n manual-test -f -

kubectl get pods -n manual-test
```

Inspect the generated YAML to see what injection actually adds:

```bash
istioctl kube-inject -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml \
  -n manual-test -o yaml | less
```

Look for the `istio-init` init container and the `istio-proxy` container — this is the diff injection makes.

---

## Step 6: Opt a Single Pod Out of Injection

Add the annotation to any deployment spec's pod template:

```yaml
metadata:
  annotations:
    sidecar.istio.io/inject: "false"
```

Apply it and confirm that pod stays at `1/1` while its neighbors stay at `2/2`.

---

## Step 7: Verify Sidecar Sync with the Control Plane

```bash
istioctl proxy-status
```

Every sidecar should show `SYNCED` for CDS, LDS, EDS, RDS. If a proxy shows `STALE`, it hasn't received the latest config push from istiod yet — check istiod logs:

```bash
kubectl logs -n istio-system deploy/istiod
```

---

## Step 8: Inspect What Envoy Actually Received

Pick a pod name from Step 4:

```bash
kubectl get pods -n bookinfo

istioctl proxy-config cluster <productpage-pod-name> -n bookinfo
istioctl proxy-config listener <productpage-pod-name> -n bookinfo
istioctl proxy-config endpoint <productpage-pod-name> -n bookinfo
```

This confirms the sidecar has real routing/cluster config pushed down from istiod via xDS — not just a bare proxy.

---

## Step 9: Confirm mTLS Is Active Between Sidecars

```bash
istioctl authn tls-check <productpage-pod-name>.bookinfo
```

Or, more directly, check for the mutual TLS handshake in the proxy config:

```bash
istioctl proxy-config secret <productpage-pod-name> -n bookinfo
```

You should see certificates issued for the workload's service account identity.

---

## Step 10: Access the App

```bash
kubectl port-forward -n bookinfo svc/productpage 9080:9080
```

Open `http://localhost:9080/productpage` in a browser — the Bookinfo app should load, proving traffic flowed through both sidecars successfully.

---

## Cleanup

```bash
kubectl delete namespace bookinfo manual-test
istioctl uninstall --purge -y
kubectl delete namespace istio-system
```

---

## Checkpoints (what you should be able to answer after this lab)

- [ ] What's the difference between the init container and the `istio-proxy` container in an injected pod?
- [ ] Why did pods created before the namespace label picked up no sidecar?
- [ ] What does `SYNCED` vs `STALE` mean in `istioctl proxy-status`?
- [ ] Where do the mTLS certificates on each sidecar actually come from?

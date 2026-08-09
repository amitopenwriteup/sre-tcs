# Kubernetes Troubleshooting Workshop Lab

**Goal:** Learn to diagnose and fix common Kubernetes issues by simulating real failures, reading diagnostic output, and implementing fixes.

**Prerequisites:**
- Working Kubernetes cluster (minikube, kind, or cloud-based)
- `kubectl` installed and configured
- `docker` (to build images locally if needed)
- Basic understanding of Pods, Services, and Deployments

**Estimated Time:** 2-3 hours

---

## Part 1: CrashLoopBackOff - Application Crash Diagnosis

### Problem: Pod repeatedly crashes and restarts

A CrashLoopBackOff means your Pod starts, crashes, then Kubernetes restarts it — infinitely. Let's simulate and fix this.

### Section 1.1: Simulate a CrashLoopBackOff

Create a deployment with a container that exits immediately:

```bash
# Create a namespace for this lab
kubectl create namespace troubleshooting-lab
kubectl config set-context --current --namespace=troubleshooting-lab
```

Create the failing deployment:

```yaml
# crash-loop-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-app
  namespace: troubleshooting-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-app
  template:
    metadata:
      labels:
        app: crash-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        command: ["/bin/bash", "-c"]
        args: ["echo 'Starting'; sleep 2; exit 1"]
```

Apply and observe:

```bash
kubectl apply -f crash-loop-deployment.yaml
kubectl get pods -w  # Watch the pod restart repeatedly
```

You'll see the pod status cycling: `Running` → `Terminated` → `CrashLoopBackOff` → repeat.

### Section 1.2: Diagnose the Issue

**Step 1: Check Pod Status**

```bash
kubectl get pods
```

Output shows `CrashLoopBackOff`:
```
NAME                        READY   STATUS              RESTARTS   AGE
crash-app-5d6f7c9b8-abc12   0/1     CrashLoopBackOff    5          2m
```

**Step 2: Read Events**

Events tell you what happened. They're chronological and often reveal root causes:

```bash
kubectl describe pod crash-app-5d6f7c9b8-abc12
```

Look for the `Events:` section at the bottom:

```
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  2m                default-scheduler  Successfully assigned troubleshooting-lab/crash-app-5d6f7c9b8-abc12 to minikube
  Normal   Pulling    2m                kubelet            Pulling image "nginx:latest"
  Normal   Pulled     1m                kubelet            Successfully pulled image "nginx:latest"
  Normal   Created    1m                kubelet            Created container app
  Normal   Started    1m                kubelet            Started container app
  Warning  BackOff    20s (x7 over 1m)  kubelet            Back-off restarting failed container
```

**Step 3: Read Logs**

The logs show what the application actually printed:

```bash
kubectl logs crash-app-5d6f7c9b8-abc12
```

Output:
```
Starting
```

The app started and exited. Why? The command explicitly calls `exit 1`.

**Step 4: Read Previous Logs (from crashed container)**

If the container crashed before restarting, get logs from the previous instance:

```bash
kubectl logs crash-app-5d6f7c9b8-abc12 --previous
```

Often shows error messages or stack traces from the crash.

### Section 1.3: Fix the Issue

The issue is the exit command. Replace it with a working application:

```yaml
# crash-loop-deployment-fixed.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-app
  namespace: troubleshooting-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-app
  template:
    metadata:
      labels:
        app: crash-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        # No command override — nginx runs as intended
```

Apply the fix:

```bash
kubectl apply -f crash-loop-deployment-fixed.yaml
kubectl get pods -w
# Pod should now stay in Running state
```

Verify:

```bash
kubectl logs crash-app-<pod-id>
# Should show nginx startup messages, no crashes
```

### Section 1.4: Troubleshooting Checklist for CrashLoopBackOff

| Question | Command | What to Look For |
|----------|---------|------------------|
| Why did it crash? | `kubectl logs <pod> --previous` | Error messages, stack traces, assertion failures |
| When did it crash? | `kubectl describe pod <pod>` → Events | Timing pattern (immediate? after delay?) |
| Did the image pull? | `kubectl describe pod <pod>` → Events | `Pulling`, `Pulled`, or `Failed to pull` messages |
| What command runs? | `kubectl get pod <pod> -o yaml` | `spec.containers[].command` and `args` |
| Health check failing? | `kubectl get pod <pod> -o yaml` → `livenessProbe` | Is a probe configured? Is it too aggressive? |

---

## Part 2: ImagePullBackOff - Container Image Problems

### Problem: Kubernetes can't pull the container image

Common causes: wrong image name, image doesn't exist, registry credentials missing, registry unreachable.

### Section 2.1: Simulate ImagePullBackOff

Create a deployment referencing a non-existent image:

```yaml
# image-pull-backoff-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-pull-app
  namespace: troubleshooting-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: image-pull-app
  template:
    metadata:
      labels:
        app: image-pull-app
    spec:
      containers:
      - name: app
        image: nonexistent-registry.io/fake-image:v99.0.0
        imagePullPolicy: Always  # Force pull attempt every time
```

Apply:

```bash
kubectl apply -f image-pull-backoff-deployment.yaml
kubectl get pods -w
```

Pod will show `ImagePullBackOff` immediately.

### Section 2.2: Diagnose the Issue

**Step 1: Check Pod Status**

```bash
kubectl get pods
```

Output:
```
NAME                             READY   STATUS              RESTARTS   AGE
image-pull-app-5d6f7c9b8-xyz98   0/1     ImagePullBackOff    0          1m
```

Notice `RESTARTS: 0` — the container never started, so it never crashed.

**Step 2: Read Events**

```bash
kubectl describe pod image-pull-app-5d6f7c9b8-xyz98
```

Events section shows:

```
Events:
  Type     Reason                 Age               From               Message
  ----     ------                 ----              ----               -------
  Normal   Scheduled              1m                default-scheduler  Successfully assigned troubleshooting-lab/image-pull-app-5d6f7c9b8-xyz98 to minikube
  Normal   BackOff                30s (x4 over 1m)  kubelet            Back-off pulling image "nonexistent-registry.io/fake-image:v99.0.0"
  Warning  Failed                 0s (x4 over 1m)  kubelet            Error: image pull failed for "nonexistent-registry.io/fake-image:v99.0.0", this may be because there are no credentials on this node. Details: (...)
```

Key clues:
- `Error: image pull failed` — image doesn't exist or is unreachable
- `no credentials on this node` — might be a private registry issue

**Step 3: Test Image Manually**

Try to pull the image locally:

```bash
docker pull nonexistent-registry.io/fake-image:v99.0.0
# Will fail with "unknown repository" or network error
```

Or via `crictl` (container runtime interface) on the node:

```bash
kubectl debug node/<node-name> -it --image=ubuntu
# Inside debug container:
crictl pull nonexistent-registry.io/fake-image:v99.0.0
```

### Section 2.3: Fix the Issue

Replace with a real, available image:

```yaml
# image-pull-deployment-fixed.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-pull-app
  namespace: troubleshooting-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: image-pull-app
  template:
    metadata:
      labels:
        app: image-pull-app
    spec:
      containers:
      - name: app
        image: nginx:latest  # Real, public image
        imagePullPolicy: IfNotPresent  # Only pull if not cached
```

Apply:

```bash
kubectl apply -f image-pull-deployment-fixed.yaml
kubectl get pods -w
# Pod should transition to Running
```

### Section 2.4: Troubleshooting Checklist for ImagePullBackOff

| Scenario | Fix |
|----------|-----|
| Image name typo | Correct the image tag in deployment YAML |
| Image doesn't exist | Use `docker search` or registry UI to find correct image |
| Private registry, no credentials | Create `docker-registry` Secret and add to `imagePullSecrets` |
| Registry unreachable from cluster | Check network policies, firewall, DNS resolution |
| Outdated image, new layer added | Use `imagePullPolicy: Always` to force fresh pull |
| Using old image tag after redeploy | Use semantic versioning, not `:latest` in prod |

Example with private registry credentials:

```bash
# Create a secret for private registry
kubectl create secret docker-registry my-registry-secret \
  --docker-server=myregistry.azurecr.io \
  --docker-username=myusername \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

Then reference it in the Pod:

```yaml
spec:
  imagePullSecrets:
  - name: my-registry-secret
  containers:
  - name: app
    image: myregistry.azurecr.io/myapp:v1.0.0
```

---

## Part 3: OOMKilled (Out of Memory)

### Problem: Container runs out of memory and gets killed

When memory usage exceeds the limit, the kernel OOM killer terminates the container.

### Section 3.1: Simulate OOMKilled

Create a deployment with a memory-hungry process and a low memory limit:

```yaml
# oom-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-hog
  namespace: troubleshooting-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: memory-hog
  template:
    metadata:
      labels:
        app: memory-hog
    spec:
      containers:
      - name: app
        image: ubuntu:latest
        command: ["/bin/bash", "-c"]
        args: ["python3 -c \"import itertools; x = list(range(10**9))\" || sleep 3600"]
        resources:
          requests:
            memory: "128Mi"
          limits:
            memory: "128Mi"  # Only 128 MB available
```

Apply:

```bash
kubectl apply -f oom-deployment.yaml
kubectl get pods -w
```

Pod will run for a few seconds then get killed.

### Section 3.2: Diagnose the Issue

**Step 1: Check Pod Status**

```bash
kubectl get pods
```

Output:
```
NAME                        READY   STATUS      RESTARTS   AGE
memory-hog-5d6f7c9b8-def45  0/1     OOMKilled   3          1m
```

Status explicitly says `OOMKilled`.

**Step 2: Read Events and Describe**

```bash
kubectl describe pod memory-hog-5d6f7c9b8-def45
```

Events:

```
Events:
  Type     Reason     Age               From      Message
  ----     ------     ----              ----      -------
  Normal   Scheduled  1m                scheduler  Successfully assigned troubleshooting-lab/memory-hog-5d6f7c9b8-def45 to minikube
  Normal   Pulled     1m                kubelet   Successfully pulled image "ubuntu:latest"
  Normal   Created    1m                kubelet   Created container app
  Normal   Started    1m                kubelet   Started container app
  Warning  OOMKilled  45s (x2 over 1m)  kubelet   Memory limit exceeded: kill -9 5432
```

Last-State shows the kill signal:

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137  # SIGKILL, always 137 for OOM
  Signal:       9
```

**Step 3: Check Current Memory Usage**

View metrics for all pods:

```bash
kubectl top pod
```

Output (if metrics-server is installed):

```
NAME               CPU(cores)   MEMORY(bytes)
memory-hog-...     1m           128Mi
```

View memory requests vs limits:

```bash
kubectl get pod memory-hog-5d6f7c9b8-def45 -o yaml | grep -A 10 resources:
```

Output:

```yaml
resources:
  limits:
    memory: 128Mi
  requests:
    memory: 128Mi
```

### Section 3.3: Fix the Issue (3 Approaches)

**Option 1: Increase Memory Limit**

If the app legitimately needs more memory:

```yaml
# oom-deployment-fixed.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-hog
  namespace: troubleshooting-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: memory-hog
  template:
    metadata:
      labels:
        app: memory-hog
    spec:
      containers:
      - name: app
        image: ubuntu:latest
        command: ["/bin/bash", "-c"]
        args: ["sleep 3600"]  # Safe command
        resources:
          requests:
            memory: "256Mi"
          limits:
            memory: "512Mi"  # Increased limit
```

**Option 2: Optimize the Application**

Reduce memory consumption in the app:

```yaml
# Lightweight app instead of memory hog
spec:
  containers:
  - name: app
    image: alpine:latest  # Smaller base image (5MB vs 77MB)
    resources:
      requests:
        memory: "64Mi"  # Reduced request
      limits:
        memory: "128Mi"
```

**Option 3: Add HPA with Memory Metrics**

If the workload is variable, auto-scale based on memory:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: memory-hog-hpa
  namespace: troubleshooting-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: memory-hog
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

Apply the fix:

```bash
kubectl apply -f oom-deployment-fixed.yaml
kubectl get pods -w
# Pod should now run without being killed
```

### Section 3.4: Troubleshooting Checklist for OOMKilled

| Question | Command | Action |
|----------|---------|--------|
| Why OOMKilled? | `kubectl describe pod <pod>` → `Exit Code: 137` | Confirm OOM (always 137) |
| Current memory usage? | `kubectl top pod <pod>` | See actual vs limit |
| Memory request/limit set? | `kubectl get pod <pod> -o yaml \| grep -A 5 resources` | Check if undersized |
| App leaking memory? | `kubectl logs <pod> --previous` | Look for memory leak warnings |
| How much to increase? | Monitor peak usage over time | Set limit 20-30% above peak |
| Multiple OOMKilled pods? | `kubectl top nodes` | Cluster memory pressure? |

---

## Part 4: Service and DNS Issues

### Problem: Pods can't reach Services, DNS resolution fails

### Section 4.1: Simulate a Service DNS Issue

Create a deployment and a service that doesn't match:

```yaml
# web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: troubleshooting-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Create a service with mismatched selectors:

```yaml
# web-service-broken.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: troubleshooting-lab
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: webapp  # WRONG! Deployment uses 'app: web'
    tier: frontend
```

Apply both:

```bash
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service-broken.yaml
```

Now create a test pod that tries to reach the service:

```yaml
# test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-client
  namespace: troubleshooting-lab
spec:
  containers:
  - name: test
    image: busybox:latest
    command: ["sleep", "3600"]
```

Apply and test:

```bash
kubectl apply -f test-pod.yaml
kubectl exec -it test-client -- sh

# Inside the pod:
wget -O- http://web.troubleshooting-lab.svc.cluster.local
# Connection refused or timeout
```

### Section 4.2: Diagnose Service Issues

**Step 1: Verify Service Exists**

```bash
kubectl get svc
```

Output:
```
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.98.200.150   <none>        80/TCP    2m
```

Service exists. Good. But are there endpoints?

**Step 2: Check Endpoints**

```bash
kubectl get endpoints web
```

Output:
```
NAME   ENDPOINTS   AGE
web    <none>      2m
```

ENDPOINTS is empty! No pods are backing this service. This is the problem.

**Step 3: Verify Selectors Match**

Get the service selector:

```bash
kubectl get svc web -o yaml | grep -A 3 selector:
```

Output:
```yaml
selector:
  app: webapp
  tier: frontend
```

Get pod labels:

```bash
kubectl get pods --show-labels
```

Output:
```
NAME                   READY   STATUS    RESTARTS   AGE    LABELS
web-5d6f7c9b8-abc12    1/1     Running   0          3m     app=web,tier=frontend
web-5d6f7c9b8-xyz98    1/1     Running   0          3m     app=web,tier=frontend
```

Mismatch: Service selector says `app=webapp` but pods have `app=web`.

**Step 4: Test DNS Directly**

```bash
kubectl exec -it test-client -- nslookup web
# or
kubectl exec -it test-client -- nslookup web.troubleshooting-lab.svc.cluster.local
```

DNS resolves to the ClusterIP, but the service has no endpoints, so connections fail.

### Section 4.3: Fix Service Issues

**Fix 1: Correct Selector**

```yaml
# web-service-fixed.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: troubleshooting-lab
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: web  # FIXED! Now matches deployment
    tier: frontend
```

Apply:

```bash
kubectl apply -f web-service-fixed.yaml
kubectl get endpoints web
# Should now show the pod IPs
```

Test from client:

```bash
kubectl exec -it test-client -- wget -O- http://web
# Should succeed and return nginx welcome page
```

**Fix 2: Verify Port Names and TargetPort**

Service must also match the container port:

```yaml
# Correct port definition
ports:
- name: http
  port: 80           # Port exposed by service
  targetPort: 8080   # Port on container (if not 80)
  protocol: TCP
```

Verify container is actually listening on targetPort:

```bash
kubectl exec -it <pod> -- netstat -tuln | grep LISTEN
```

### Section 4.4: Debugging Service and DNS

Create a comprehensive debug manifest:

```yaml
# service-debug-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: troubleshooting-lab
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot:latest  # Packed with networking tools
    command: ["sleep", "3600"]
```

Apply and run diagnostics:

```bash
kubectl apply -f service-debug-pod.yaml
kubectl exec -it debug-pod -- sh
```

Inside the debug pod, run these commands:

```bash
# Resolve DNS
nslookup web
nslookup web.troubleshooting-lab.svc.cluster.local
dig web.troubleshooting-lab.svc.cluster.local

# Test connectivity
curl http://web:80
curl http://10.98.200.150:80  # ClusterIP directly

# Check routing
traceroute web
mtr -r -c 5 web

# See what ports are open
nc -zv web 80

# Inspect service from outside the pod
kubectl describe svc web
kubectl get svc web -o yaml
kubectl get endpoints web
```

### Section 4.5: Troubleshooting Checklist for Service/DNS

| Issue | Command | Fix |
|-------|---------|-----|
| Service has no endpoints | `kubectl get endpoints <svc>` | Fix selector to match pod labels |
| DNS doesn't resolve | `kubectl exec <pod> -- nslookup <svc>` | Check CoreDNS pod status: `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| DNS resolves but connection fails | `kubectl exec <pod> -- curl http://<svc>` | Check service ports, targetPort, container listening port |
| Wrong port on targetPort | `kubectl describe svc <svc>` | Match targetPort to container port |
| Selector mismatch | `kubectl get pods --show-labels` + `kubectl get svc <svc> -o yaml` | Ensure labels match exactly |
| Service in wrong namespace | Check if client and service in same namespace | Use FQDN: `<svc>.<namespace>.svc.cluster.local` |
| Network policy blocking | `kubectl get networkpolicy` | Check ingress/egress rules allow traffic |

---

## Part 5: Node Pressure and Resource Issues

### Problem: Cluster runs out of resources, nodes have pressure conditions

Node pressure occurs when disk, memory, or PIDs are scarce on a node.

### Section 5.1: Simulate Node Pressure

Create several large deployments to exhaust node resources:

```yaml
# resource-pressure-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-eater
  namespace: troubleshooting-lab
spec:
  replicas: 5
  selector:
    matchLabels:
      app: resource-eater
  template:
    metadata:
      labels:
        app: resource-eater
    spec:
      containers:
      - name: app
        image: busybox:latest
        command: ["sh", "-c"]
        args: ["while true; do echo 'busy'; done"]  # CPU hog
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
```

Apply multiple times:

```bash
for i in {1..3}; do
  kubectl apply -f resource-pressure-deployment.yaml --dry-run=client -o yaml | \
    sed "s/resource-eater/resource-eater-$i/g" | kubectl apply -f -
done

kubectl get pods
# Some pods will be in Pending status
```

### Section 5.2: Diagnose Node Pressure

**Step 1: Check Node Status**

```bash
kubectl get nodes
```

Look for conditions (often not visible in short output). Get full details:

```bash
kubectl describe node <node-name>
```

Look for `Conditions` section:

```
Conditions:
  Type             Status  LastHeartbeatTime         LastTransitionTime        Reason                Message
  ----             ------  -----------------         ------------------        ------                -------
  MemoryPressure   True    2024-01-15T10:30:45Z      2024-01-15T10:25:30Z      KubeletHasMemoryPressure
  DiskPressure     False   2024-01-15T10:30:45Z      2024-01-15T10:00:00Z      KubeletHasNoDiskPressure
  PIDPressure      False   2024-01-15T10:30:45Z      2024-01-15T10:00:00Z      KubeletHasPIDPressure
  Ready            False   2024-01-15T10:30:45Z      2024-01-15T10:25:30Z      KubeletNotReady        Kubelet has MemoryPressure condition
```

`MemoryPressure: True` indicates the node is out of memory.

**Step 2: Check Resource Usage**

```bash
kubectl top nodes
kubectl top pods -A
```

Output shows which pods are consuming the most resources.

**Step 3: Check Pending Pods**

```bash
kubectl get pods --all-namespaces --field-selector=status.phase=Pending
kubectl describe pod <pending-pod-name>
```

Events section shows why pod can't be scheduled:

```
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  2m (x5 over 5m)   default-scheduler  0/1 nodes are available: 1 Insufficient memory.
```

### Section 5.3: Fix Node Pressure (Multiple Approaches)

**Fix 1: Delete Non-Essential Pods**

Free up resources immediately:

```bash
kubectl delete deployment resource-eater
kubectl delete deployment resource-eater-1 resource-eater-2 resource-eater-3
kubectl get pods --all-namespaces --field-selector=status.phase=Failed -o json | \
  kubectl delete -f -
```

Verify node recovers:

```bash
kubectl describe node <node-name> | grep Conditions: -A 10
# MemoryPressure should become False
```

**Fix 2: Increase Resource Requests/Limits**

If using minikube or single-node cluster, allocate more resources:

```bash
# For minikube:
minikube delete
minikube start --cpus=4 --memory=8192
```

For managed clusters (EKS, GKE, AKS), increase node pool size or add nodes.

**Fix 3: Add Resource Requests to Pods**

Ensure pods have realistic requests so scheduler can make smart decisions:

```yaml
# resource-aware-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-requests
  namespace: troubleshooting-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-with-requests
  template:
    metadata:
      labels:
        app: app-with-requests
    spec:
      containers:
      - name: app
        image: nginx:latest
        resources:
          requests:
            cpu: "100m"          # 100 millicores
            memory: "64Mi"       # 64 megabytes
          limits:
            cpu: "500m"
            memory: "256Mi"
```

**Fix 4: Use Quality of Service (QoS) Classes**

Kubernetes evicts pods in order: BestEffort → Burstable → Guaranteed

```yaml
# guaranteed-qos.yaml - Will be last to evict
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
  namespace: troubleshooting-lab
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "100m"  # Must equal request
        memory: "128Mi"  # Must equal request
  # When requests == limits, QoS class is Guaranteed
```

**Fix 5: Enable Kubelet Eviction Policies**

Configure kubelet to evict pods before node becomes unresponsive:

```bash
# On node (requires SSH or node access):
# Edit /etc/kubernetes/kubelet/kubelet-config.yaml or kubelet args:
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
evictionSoft:
  memory.available: "500Mi"
  nodefs.available: "20%"
evictionSoftGracePeriod:
  memory.available: "1m30s"
  nodefs.available: "2m"
```

### Section 5.4: Troubleshooting Checklist for Node Pressure

| Condition | Command | Fix |
|-----------|---------|-----|
| MemoryPressure | `kubectl describe node <node>` | Delete pods, increase memory, set requests |
| DiskPressure | `kubectl describe node <node>` | Clean logs, images; add storage |
| PIDPressure | `kubectl describe node <node>` | Reduce workload, increase PID limit |
| Pending pods | `kubectl describe pod <pod>` | Check `FailedScheduling` events |
| High memory usage | `kubectl top pod -A --sort-by=memory` | Identify resource hogs, cap with limits |
| High CPU usage | `kubectl top pod -A --sort-by=cpu` | Cap with limits, scale horizontally |
| Unschedulable nodes | `kubectl get nodes` | Add labels/taints correctly to match nodeSelectors |

---

## Part 6: Complete Lab Scenario - Multi-Issue Debugging

Combine all issues in one scenario: a deployment with mistakes in image, memory, service, and placement.

### Section 6.1: The Broken Application Stack

```yaml
# complete-lab-scenario.yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: broken-app-lab

---
# Deployment with multiple issues
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: broken-app-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
      - name: app
        image: my-fake-registry.io/app:v1  # Issue 1: Non-existent image
        resources:
          limits:
            memory: "32Mi"  # Issue 2: Way too low, will OOMKill
        ports:
        - containerPort: 8080

---
# Service with wrong selector
apiVersion: v1
kind: Service
metadata:
  name: broken-app
  namespace: broken-app-lab
spec:
  selector:
    app: broken-application  # Issue 3: Doesn't match deployment label
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP

---
# Client trying to use service
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: broken-app-lab
spec:
  containers:
  - name: client
    image: curlimages/curl
    command: ["sleep", "3600"]
```

Apply:

```bash
kubectl apply -f complete-lab-scenario.yaml
```

### Section 6.2: Diagnose All Issues

Run through the diagnostic sequence:

```bash
# Issue 1: Check pods
kubectl get pods -n broken-app-lab
# Some show ImagePullBackOff

# Issue 2: Check events
kubectl describe pod broken-app-<pod-id> -n broken-app-lab
# "Failed to pull image"

# Issue 3: Check service
kubectl get endpoints broken-app -n broken-app-lab
# No endpoints

# Issue 4: Verify selectors
kubectl get svc broken-app -n broken-app-lab -o yaml | grep selector: -A 2
kubectl get pods -n broken-app-lab --show-labels

# Issue 5: Test connectivity
kubectl exec -it client -n broken-app-lab -- curl http://broken-app
# Connection refused
```

### Section 6.3: Fix All Issues

Create corrected manifest:

```yaml
# complete-lab-scenario-fixed.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: broken-app-lab

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: broken-app-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
      - name: app
        image: nginx:latest  # Fixed: Real image
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"  # Fixed: Reasonable limit
            cpu: "500m"
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: broken-app
  namespace: broken-app-lab
spec:
  selector:
    app: broken-app  # Fixed: Matches deployment label
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP

---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: broken-app-lab
spec:
  containers:
  - name: client
    image: nicolaka/netshoot:latest
    command: ["sleep", "3600"]
```

Apply fixes:

```bash
kubectl apply -f complete-lab-scenario-fixed.yaml
kubectl get pods -n broken-app-lab -w
# Wait for all pods to reach Running

# Verify service has endpoints
kubectl get endpoints broken-app -n broken-app-lab

# Test from client
kubectl exec -it client -n broken-app-lab -- curl http://broken-app
# Should see nginx welcome page
```

---

## Part 7: Advanced Debugging Tools

### Using kubectl debug

Interactive debugging session on a pod:

```bash
# Debug a running pod with a debug container
kubectl debug <pod-name> -it --image=ubuntu

# Debug a node by spawning a privileged pod on it
kubectl debug node/<node-name> -it --image=ubuntu
```

### Using logs with filters

```bash
# Last 50 lines
kubectl logs <pod> --tail=50

# Follow logs in real-time
kubectl logs <pod> -f

# Logs from 10 minutes ago
kubectl logs <pod> --since=10m

# Logs with timestamps
kubectl logs <pod> --timestamps=true

# From specific container if pod has multiple
kubectl logs <pod> -c <container-name>
```

### Using port-forward for debugging

Forward a service to your local machine:

```bash
# Forward pod port to local port
kubectl port-forward pod/<pod-name> 8080:80

# Forward service port
kubectl port-forward svc/<service-name> 8080:80

# Listen on all interfaces
kubectl port-forward --address 0.0.0.0 svc/<service-name> 8080:80
```

Then access locally:

```bash
curl http://localhost:8080
```

### Using exec for live inspection

```bash
# Run a command in the container
kubectl exec <pod> -- ps aux
kubectl exec <pod> -- netstat -tuln
kubectl exec <pod> -- env
kubectl exec <pod> -- cat /etc/os-release

# Interactive shell
kubectl exec -it <pod> -- /bin/bash
```

### Using top for resource monitoring

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods

# By namespace
kubectl top pods -n <namespace>

# Sort by memory
kubectl top pods --sort-by=memory

# All namespaces
kubectl top pods -A
```

---

## Part 8: Cleanup

Remove all resources created in this lab:

```bash
# Delete namespaces (deletes all resources within)
kubectl delete namespace troubleshooting-lab
kubectl delete namespace broken-app-lab

# Or manually delete resources
kubectl delete deployment crash-app
kubectl delete deployment image-pull-app
kubectl delete deployment memory-hog
kubectl delete deployment web
kubectl delete svc web
kubectl delete pod test-client
```

---

## Quick Reference: Debugging Flowchart

```
Pod not Running?
├─ Status: CrashLoopBackOff
│  └─ Run: kubectl logs <pod> --previous
│     └─ Look for: exit code, error messages
│
├─ Status: ImagePullBackOff
│  └─ Run: kubectl describe pod <pod>
│     └─ Look for: "Failed to pull image"
│
├─ Status: OOMKilled
│  └─ Run: kubectl top pod <pod>
│     └─ Solution: Increase memory limit
│
├─ Status: Pending
│  └─ Run: kubectl describe pod <pod>
│     └─ Look for: "FailedScheduling"
│
└─ Status: Running but not working?
   ├─ Check Service: kubectl get endpoints <svc>
   ├─ Check DNS: kubectl exec <pod> -- nslookup <svc>
   └─ Check Connectivity: kubectl exec <pod> -- curl http://<svc>
```

---

## Common Commands Cheat Sheet

```bash
# Status and info
kubectl get pods -w                          # Watch pods
kubectl get pods -A                          # All namespaces
kubectl get events -A --sort-by='.lastTimestamp'  # Sorted events
kubectl describe pod <pod>                   # Full pod details
kubectl describe node <node>                 # Node conditions and pressure

# Logs and output
kubectl logs <pod>                           # Container logs
kubectl logs <pod> --previous                # Logs from crashed container
kubectl logs <pod> -f                        # Follow/tail logs
kubectl logs <pod> -c <container>            # Specific container
kubectl logs <pod> --tail=100                # Last 100 lines

# Inspection
kubectl get pod <pod> -o yaml                # Full YAML definition
kubectl get svc <svc> -o yaml                # Service definition
kubectl get endpoints <svc>                  # Service endpoints
kubectl exec -it <pod> -- /bin/bash          # Shell access

# Debugging
kubectl debug pod/<pod> -it --image=ubuntu   # Debug container
kubectl debug node/<node> -it                # Debug node
kubectl port-forward <pod> 8080:80           # Port forward
kubectl top pods                             # Resource usage
kubectl get nodes                            # Node status
kubectl describe node <node>                 # Node details

# Cleanup
kubectl delete pod <pod>                     # Delete pod
kubectl delete deployment <dep>              # Delete deployment
kubectl delete namespace <ns>                # Delete namespace
```

---

## Learning Outcomes

After completing this lab, you should be able to:

- [ ] Identify and diagnose CrashLoopBackOff issues using logs and events
- [ ] Recognize and fix ImagePullBackOff problems
- [ ] Spot OOMKilled containers and adjust memory limits
- [ ] Debug Service connectivity and DNS resolution issues
- [ ] Understand selectors, labels, and why services need matching endpoints
- [ ] Recognize and address node resource pressure
- [ ] Use kubectl describe, logs, exec, and debug effectively
- [ ] Read Kubernetes events to understand cluster behavior
- [ ] Build mental models of common failure modes
- [ ] Use systematic troubleshooting approaches

---

## Additional Resources

- [Kubernetes Debugging Guide](https://kubernetes.io/docs/tasks/debug/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Troubleshooting Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Troubleshooting Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Pod Lifecycle Events](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

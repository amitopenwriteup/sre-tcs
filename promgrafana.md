# Prometheus Installation Lab Guide

## Setting Up Prometheus, Grafana, and Loki on Kubernetes

**Created:** 2024-2025  
**Purpose:** Comprehensive lab guide covering the installation and configuration of Prometheus monitoring stack on Kubernetes clusters using Helm charts.

---

## Table of Contents

1. [Prerequisites and Overview](#prerequisites-and-overview)
2. [Prometheus Installation](#prometheus-installation)
3. [Configuring PersistentVolumeClaims (PVCs)](#configuring-persistentvolumeclaims-pvcs)
4. [Service Configuration and Access](#service-configuration-and-access)
5. [Grafana Installation and Setup](#grafana-installation-and-setup)
6. [Connecting Grafana to Prometheus](#connecting-grafana-to-prometheus)
7. [Loki Setup and Log Aggregation (SingleBinary Mode)](#loki-setup-and-log-aggregation)
   - 7.1 Prerequisites for Loki
   - 7.2 Create Configuration Files
   - 7.3 Prepare Worker Node Storage
   - 7.4 Deploy Loki
   - 7.5 Verify Installation
   - 7.6 Access Loki
   - 7.7 Verify Service
   - 7.8 Test Connectivity
   - 7.9 Check Memory Usage
   - 7.10 Add to Grafana
   - 7.11 Query Logs
   - 7.12 Architecture Overview
   - 7.13 Troubleshooting
   - 7.14 Uninstall
   - 7.15 Production Considerations
8. [Complete Stack: Troubleshooting and Uninstall](#complete-stack-troubleshooting-and-uninstall)

---

## Prerequisites and Overview

Before starting this lab, ensure you have the following:

- A running Kubernetes cluster
- `kubectl` configured to access your cluster
- Helm 3.x installed on your machine
- NFS or other persistent storage available (optional but recommended)
- Basic understanding of Kubernetes concepts

### What We'll Accomplish

In this lab, we will:

- Install Prometheus using Helm charts
- Configure persistent storage for metrics
- Install and configure Grafana for visualization
- Integrate Loki for log aggregation

---

## Prometheus Installation

### 2.1 Search for Prometheus Helm Chart

Search for available Prometheus charts in the Helm Hub:

```bash
helm search hub Prometheus
```

You can also browse available charts online at:  
**https://artifacthub.io/**

### 2.2 Add Prometheus Helm Repository

Add the Prometheus community Helm chart repository:

```bash
helm repo add prometheus-community \
    https://prometheus-community.github.io/helm-charts
```

Update the repository index:

```bash
helm repo update
```

### 2.3 Install Prometheus Helm Chart

Deploy Prometheus to your Kubernetes cluster:

```bash
helm install prometheus prometheus-community/prometheus
```

### 2.4 Verify the Installation

Check that all pods and storage resources have been created:

```bash
kubectl get pods
kubectl get pvc
```

**Expected output:** You should see Prometheus server pods and storage claims created in the default namespace.

---

## Configuring PersistentVolumeClaims (PVCs)

To ensure data persistence, configure the PVCs with a storage class (e.g., nfs-client).

### 3.1 Edit Prometheus Server PVC

Update the Prometheus server PVC storage class:

```bash
kubectl edit pvc prometheus-server
```

Add or modify the following in the `.spec` section:

```yaml
spec:
  storageClassName: nfs-client
```

### 3.2 Edit AlertManager PVC

Update the AlertManager PVC storage class:

```bash
kubectl edit pvc prometheus-alertmanager-0
```

Add the same storage class configuration:

```yaml
spec:
  storageClassName: nfs-client
```

### Important Notes

Ensure the storage class `nfs-client` exists in your cluster. You can verify with:

```bash
kubectl get storageclass
```

If the storage class doesn't exist, you may need to install a provisioner or create a custom storage class for your environment.

---

## Service Configuration and Access

### 4.1 Port Forwarding (Temporary Access)

To access Prometheus locally via port forwarding:

```bash
kubectl port-forward --address 0.0.0.0 \
    svc/prometheus-server 9090:80
```

Access Prometheus at: **http://localhost:9090**

### 4.2 View Current Services

List all services to see their current configuration:

```bash
kubectl get service
```

### 4.3 Convert to NodePort (Persistent Access)

For persistent access, convert the service from ClusterIP to NodePort:

```bash
kubectl edit svc prometheus-server
```

Change the service type to NodePort:

```yaml
spec:
  type: NodePort
```

After saving, note the assigned NodePort and access Prometheus via:

```
http://<node-ip>:<nodeport>
```

---

## Grafana Installation and Setup

### 5.1 Search for Grafana Helm Chart

Search for available Grafana charts:

```bash
helm search hub Grafana
```

### 5.2 Add Grafana Helm Repository

Add the official Grafana Helm repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update the repository:

```bash
helm repo update
```

### 5.3 Install Grafana

Deploy Grafana to your cluster:

```bash
helm install grafana grafana/grafana
```

### 5.4 Get Grafana Admin Password

Retrieve the auto-generated admin password:

```bash
kubectl get secret --namespace default grafana \
    -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```



### 5.5 Edit Grafana Service Type

Convert Grafana service to NodePort for external access:

```bash
kubectl edit svc grafana
```

Change the service type to NodePort and note the assigned port.

---

## Connecting Grafana to Prometheus

### 6.1 Access Grafana Portal

Open Grafana in your browser using the NodePort address. Log in with:

- **Username:** `admin`
- **Password:** `<password from step 5.4>`

### 6.2 Add Prometheus Data Source

Steps to configure Prometheus as a data source:

1. Click on **Configuration** (gear icon) in the left menu
2. Select **Data Sources**
3. Click **Add data source**
4. Select **Prometheus** from the list
5. In the URL field, enter: `http://prometheus-server:80`
6. Click **Save & Test** to verify the connection

### 6.3 Create Dashboards

You can now:
- Create custom dashboards using the Prometheus metrics
- Import pre-built Prometheus dashboards from Grafana's dashboard library
- Build alerts and notification channels for proactive monitoring

---

## Loki Setup and Log Aggregation

Grafana Loki is a horizontally scalable, highly available log aggregation system inspired by Prometheus. It's designed to be cost-effective and easy to operate. In this lab, we'll deploy Loki in SingleBinary mode, which runs all components in a single container - perfect for testing and small production environments.

### 7.1 Prerequisites for Loki Setup

Before installing Loki, ensure you have:

- At least two Kubernetes nodes (one master, one worker)
- kubectl configured to access your cluster
- Helm 3.x installed
- SSH access to your worker node for storage setup

### 7.2 Step 1: Create Configuration Files

Create a working directory on your master node:

```bash
mkdir -p ~/loki-setup
cd ~/loki-setup
```

Create `loki-values.yaml` with reduced memory settings:

```bash
cat > loki-values.yaml <<'EOF'
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
    ring:
      kvstore:
        store: inmemory
  
  schemaConfig:
    configs:
      - from: "2023-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  
  storage:
    type: filesystem
  
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 256Mi
  
  pattern_ingester:
    enabled: true
  
  limits_config:
    ingestion_rate_mb: 8
    ingestion_burst_size_mb: 16
    max_global_streams_per_user: 5000
    reject_old_samples: true
    reject_old_samples_max_age: 168h
    retention_period: 168h
    allow_structured_metadata: true
    volume_enabled: true
  
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: manual

deploymentMode: SingleBinary

singleBinary:
  replicas: 1

backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0

service:
  type: NodePort
  port: 3100
  nodePort: 31000

readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 30
  timeoutSeconds: 1
  
livenessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 45
  timeoutSeconds: 1

securityContext:
  fsGroup: 10001
  runAsUser: 10001
  runAsGroup: 10001

canary:
  enabled: false
gateway:
  enabled: false
results_cache:
  enabled: false
chunks_cache:
  enabled: false
memcached:
  enabled: false

memberlist:
  service:
    enabled: false

promtail:
  enabled: false
grafana:
  enabled: false
prometheus:
  enabled: false
EOF
```

Create `pv-loki.yaml` for persistent storage with manual storage class:

```bash
cat > pv-loki.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: storage-loki-0
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  claimRef:
    namespace: loki
    name: storage-loki-0
  hostPath:
    path: "/mnt/data/loki"
    type: DirectoryOrCreate
EOF
```

### 7.3 Step 2: Prepare Storage on Worker Node

Before deploying, ensure your cluster has the "manual" storage class. Check if it exists:

```bash
kubectl get storageclass manual
```

If it doesn't exist, create it:

```bash
kubectl apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
```

SSH into your worker node and set up the storage directory:

```bash
# SSH to worker node
ssh <worker-node-ip>

# Clean installation
sudo rm -rf /mnt/data/loki
sudo mkdir -p /mnt/data/loki

# Set ownership and permissions (match securityContext in values)
sudo chown -R 10001:10001 /mnt/data/loki
sudo chmod -R 0700 /mnt/data/loki

# Verify
ls -la /mnt/data/loki

# Exit SSH
exit
```

### 7.4 Step 3: Deploy Loki

Add Grafana Helm repository and create namespace:

```bash
# Add Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create namespace for Loki
kubectl create namespace loki

# Apply the PersistentVolume
kubectl apply -f pv-loki.yaml
```

Deploy Loki using Helm:

```bash
helm install loki grafana/loki \
  --namespace loki \
  --values loki-values.yaml \
  --version 6.29.0
```

### 7.5 Step 4: Verify the Installation

Check PVC status:

```bash
kubectl get pvc -n loki
```

If PVC status is **Pending**, verify and reapply the PersistentVolume:

```bash
kubectl delete pv storage-loki-0
kubectl apply -f pv-loki.yaml

# Wait and check again
sleep 10
kubectl get pvc -n loki
```

Check pod status:

```bash
kubectl get pods -n loki -w
```

Expected output: You should see `loki-0` pod in **Running** state with **1/1** ready containers.

### 7.6 Step 5: Access Loki

Get your worker node IP:

```bash
kubectl get nodes -o wide
```

Access Loki at:

```
http://<worker-node-ip>:31000
```

Or using port-forward:

```bash
kubectl port-forward -n loki svc/loki 3100:3100
```

### 7.7 Verify Loki Service

Check service configuration:

```bash
kubectl get svc -n loki
kubectl describe svc loki -n loki
```

If needed, patch the service:

```bash
kubectl patch svc loki -n loki -p '{"spec": {"type": "NodePort", "ports": [{"port": 3100, "targetPort": "http", "nodePort": 31000, "name": "http"}]}}'
```

### 7.8 Test Loki Connectivity

Test from within the cluster:

```bash
kubectl run -it --rm test -n loki --image=curlimages/curl --restart=Never -- \
  curl http://loki:3100/ready
```

Expected output: Should return HTTP 200 OK

### 7.9 Check Memory Usage

Verify reduced memory consumption:

```bash
kubectl top pod -n loki
```

You should see memory usage around 64-256Mi.

### 7.10 Add Loki Data Source to Grafana

Once Loki is running, configure it in Grafana:

1. Go to Grafana (running on Prometheus namespace or separate Grafana instance)
2. Click **Configuration** (gear icon)
3. Select **Data Sources**
4. Click **Add data source**
5. Select **Loki**
6. In URL field, enter: `http://loki.loki:3100` (cross-namespace)
7. Click **Save & Test**

### 7.11 Query Logs in Grafana

Once Loki is connected:

1. Go to **Explore**
2. Select **Loki** data source
3. Use LogQL to query:

Example queries:

```logql
{namespace="loki"}
```

```logql
{job=~".+"} |= "error"
```

```logql
count_over_time({namespace="loki"}[5m])
```

### 7.12 SingleBinary Mode Architecture

This deployment uses:

- **Deployment Mode:** SingleBinary (all components in one pod)
- **Storage:** Filesystem with 10GB persistent volume using "manual" storage class
- **Storage Path:** /mnt/data/loki on worker node (hostPath)
- **Retention:** 7 days (168 hours)
- **Memory:** 64Mi request, 256Mi limit (lightweight)
- **Log Ingestion:** 8MB/s rate limiting
- **Access:** NodePort on port 31000
- **Namespace:** loki (isolated)

### 7.13 Troubleshooting Loki

**Issue:** PVC stuck in Pending state

**Solution:** Check if storage class "manual" exists and if PersistentVolume is available:
```bash
kubectl get storageclass manual
kubectl get pv storage-loki-0
kubectl describe pv storage-loki-0
kubectl get pvc -n loki
```

If storage class doesn't exist, create it:
```bash
kubectl apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
```

If PV is not bound, reapply it:
```bash
kubectl delete pv storage-loki-0
kubectl apply -f pv-loki.yaml
```

---

**Issue:** Pod not starting

**Solution:** Check pod logs:
```bash
kubectl logs -n loki -f svc/loki
```

---

**Issue:** High memory usage

**Solution:** Reduce retention period in loki-values.yaml:
```yaml
retention_period: 24h  # Instead of 168h
```

---

**Issue:** No logs appearing

**Solution:** Ensure log collectors are properly configured to send logs to:
```
http://loki.loki:3100/loki/api/v1/push
```

### 7.14 Uninstall Loki

To remove Loki from your cluster:

```bash
helm uninstall loki -n loki

# Delete PVC
kubectl delete pvc -n loki storage-loki-0

# Delete namespace
kubectl delete namespace loki
```

### 7.15 Production Considerations

For production environments, consider:

- Enable authentication (`auth_enabled: true`)
- Use distributed storage (not hostPath)
- Scale to multiple replicas with proper HA setup
- Implement log shippers (Promtail, Fluent Bit)
- Configure persistent backups
- Set up proper retention policies
- Use reverse proxy with TLS

**Official Documentation:**
https://grafana.com/docs/loki/latest/setup/install/

---

## Complete Stack: Troubleshooting and Uninstall

### 8.1 Complete Monitoring Stack Overview

Your complete monitoring and logging stack now includes:

- **Prometheus:** Collects and stores metrics from your applications and infrastructure
- **Grafana:** Provides visualization dashboards for metrics from Prometheus and logs from Loki
- **Loki (SingleBinary):** Aggregates and indexes logs using "manual" storage class for persistent storage
- **Storage:** Uses hostPath-backed manual storage class for both Prometheus and Loki

This integrated stack provides complete observability for your Kubernetes cluster, combining metrics and logs in a single Grafana interface.

### 8.2 Useful Kubectl Commands

Check pod status:

```bash
kubectl get pods -n default
kubectl describe pod <pod-name>
```

View pod logs:

```bash
kubectl logs <pod-name>
kubectl logs -l app=prometheus
kubectl logs -l app=grafana
kubectl logs -l app=loki
```

Check service status:

```bash
kubectl get services
kubectl describe svc <service-name>
```

Edit resources:

```bash
kubectl edit deployment <deployment-name>
kubectl edit configmap <configmap-name>
```

### 8.3 Uninstall Helm Releases

If you need to remove all components of the monitoring stack:

```bash
# Uninstall in this order
helm uninstall loki
helm uninstall prometheus
helm uninstall grafana
```

**Note:** Remove Loki first to ensure Promtail is cleaned up properly before removing the monitoring cluster.

### 8.4 Clean Up Persistent Volumes

After uninstalling, you may need to manually delete PVCs:

```bash
# List PVCs
kubectl get pvc

# Delete specific PVC
kubectl delete pvc <pvc-name>

# Delete all PVCs in default namespace
kubectl delete pvc --all
```

**Warning:** Deleting PVCs will remove all stored data. Ensure you have backups if needed.

### 8.5 Complete Stack: Common Issues and Solutions

#### Issue: Pods remain in Pending state

**Solution:** Check if PVs are available or if resource limits are met:
```bash
kubectl describe pod <pod-name>
kubectl get pv
kubectl describe pv <pv-name>
```

---

#### Issue: Prometheus cannot scrape metrics

**Solution:** Verify network policies and service discovery configuration:
```bash
kubectl logs -l app=prometheus-server
kubectl get configmap prometheus-server -o yaml
```

---

#### Issue: Grafana cannot connect to Prometheus

**Solution:** Check DNS resolution and ensure the data source URL is correct:
```bash
# Test connectivity from Grafana pod
kubectl exec -it <grafana-pod> -- curl http://prometheus-server:80

# Check service endpoints
kubectl get endpoints prometheus-server
```

---

#### Issue: Promtail pods in CrashLoopBackOff state

**Solution:** Check Promtail configuration for syntax errors and verify Loki service accessibility:
```bash
kubectl logs -l app=promtail --tail=100
kubectl describe pod -l app=promtail
```

---

#### Issue: LogQL queries returning no results

**Solution:** Verify that Promtail is actually scraping logs; check pod labels match your query:
```bash
# Check available labels in Loki
# In Grafana Explore, use the label browser
# Or query: {job=~".+"}  (to list all jobs)
```

---

#### Issue: Combined metrics and logs dashboard not showing

**Solution:** Ensure both Prometheus and Loki data sources are properly configured and tested in Grafana:
```bash
# In Grafana, go to Configuration > Data Sources
# Test both Prometheus and Loki connections
# Verify URLs are accessible from Grafana pod
```

---

#### Issue: Storage issues with Loki

**Solution:** Configure persistent storage for Loki or adjust retention policies to manage disk space:
```bash
# Check Loki storage
kubectl get pvc
kubectl describe pvc loki

# Edit Loki values to adjust retention
helm get values loki
helm upgrade loki grafana/loki-stack --set retention_period=24h
```

---

#### Issue: High CPU or memory usage in the monitoring stack

**Solution:** Optimize scrape intervals and retention policies:
```bash
# For Prometheus - increase scrape interval
helm upgrade prometheus prometheus-community/prometheus \
    --set prometheus.prometheusSpec.scrapeInterval=30s

# For Loki - adjust chunk cache and retention
helm upgrade loki grafana/loki-stack \
    --set loki.config.limits_config.retention_period=24h
```

---

## Monitoring Best Practices

### 1. Scrape Interval Configuration
- Default is 30 seconds - adjust based on your needs
- Larger intervals = less storage, but less granular data
- Smaller intervals = more storage, more detailed data

### 2. Retention Policies
- Prometheus: Configure retention with `--storage.tsdb.retention.time` flag
- Loki: Set retention period in Loki configuration
- Balance between storage costs and data availability

### 3. Resource Allocation
- Monitor CPU and memory usage of monitoring stack
- Allocate sufficient resources for the volume of metrics/logs you're collecting
- Use HPA (Horizontal Pod Autoscaling) for scalability

### 4. Alerting Setup
- Configure alert rules in Prometheus for critical metrics
- Set up notification channels in Grafana (Slack, PagerDuty, email, etc.)
- Test alerts regularly to ensure they work

### 5. Log Management
- Filter logs at the source with Promtail configuration
- Use appropriate retention periods
- Archive old logs if needed for compliance

---

## Useful Resources

- **Prometheus Documentation:** https://prometheus.io/docs/
- **Grafana Documentation:** https://grafana.com/docs/grafana/
- **Loki Documentation:** https://grafana.com/docs/loki/
- **Helm Charts:** https://artifacthub.io/
- **Kubernetes Documentation:** https://kubernetes.io/docs/

---

## Quick Reference: Command Summary

### Prometheus
```bash
# Install
helm install prometheus prometheus-community/prometheus

# Uninstall
helm uninstall prometheus

# Port forward
kubectl port-forward svc/prometheus-server 9090:80
```

### Grafana
```bash
# Install
helm install grafana grafana/grafana

# Get password
kubectl get secret grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Uninstall
helm uninstall grafana

# Port forward
kubectl port-forward svc/grafana 3000:80
```

### Loki (SingleBinary Mode)

Note: This setup uses the "manual" storage class. Ensure it exists before deployment.

```bash
# Verify or create manual storage class
kubectl get storageclass manual || \
kubectl apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

# Setup
mkdir -p ~/loki-setup
cd ~/loki-setup

# Create values file (see section 7.2)
cat > loki-values.yaml <<'EOF'
# [Use values from section 7.2]
EOF

# Create PersistentVolume (see section 7.2)
cat > pv-loki.yaml <<'EOF'
# [Use PV config from section 7.2 with storageClassName: manual]
EOF

# Deploy
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
kubectl create namespace loki
kubectl apply -f pv-loki.yaml
helm install loki grafana/loki \
  --namespace loki \
  --values loki-values.yaml \
  --version 6.29.0

# Check status
kubectl get pods -n loki
kubectl get pvc -n loki
kubectl get storageclass manual
kubectl logs -n loki svc/loki

# Access
# http://<worker-node-ip>:31000

# Uninstall
helm uninstall loki -n loki
kubectl delete pvc -n loki storage-loki-0
kubectl delete namespace loki
```

---

## Document Information

- **Last Updated:** 2024-2025
- **Format:** Markdown
- **Lab Duration:** 30-45 minutes
- **Difficulty Level:** Intermediate
- **Prerequisites:** Kubernetes fundamentals, Helm basics

---

**For additional information and updates, consult the official Prometheus, Grafana, and Loki documentation.**

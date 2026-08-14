# Kubernetes Upgrade Workshop: v1.30 → v1.35.6
## Complete Step-by-Step Guide using kubeadm on Rocky Linux 9 with Flannel

### Current Cluster Status
- **Current Version**: v1.30.14
- **Target Version**: v1.35.6
- **Upgrade Method**: kubeadm (incremental)
- **Networking**: Flannel (VXLAN backend)
- **Operating System**: Rocky Linux 9
- **Total Upgrade Cycles**: 5 minor version increments

---

## Table of Contents
1. [Version Compatibility & Repository Setup](#version-compatibility--repository-setup)
2. [Pre-Upgrade Checklist](#pre-upgrade-checklist)
3. [Preparation & Planning](#preparation--planning)
4. [Control Plane Node Upgrade](#control-plane-node-upgrade)
5. [Worker Node Upgrades](#worker-node-upgrades)
6. [Verification & Testing](#verification--testing)
7. [Troubleshooting](#troubleshooting)
8. [Rollback Procedures](#rollback-procedures)

---

## Version Compatibility & Repository Setup

### ⚠️ IMPORTANT: Kubernetes Version Constraints

**Kubernetes only allows upgrading across 1 minor version at a time.**

```
Example: 1.30 → 1.31 → 1.32 → 1.33 → 1.34 → 1.35
Cannot skip: 1.30 → 1.35 (NOT ALLOWED - will fail)
```

### Step 1: Check Current Kubernetes Version

```bash
# Check what version you're running
kubeadm version
kubelet --version
kubectl version --short

# Example output:
# kubeadm version: v1.30.14
# Kubelet v1.30.14
# Client Version: v1.30.14
```

### Step 2: Identify Your Current Version

If you're at **1.30.x**, your upgrade path is:
```
1.30.14 → 1.31.x → 1.32.x → 1.33.x → 1.34.x → 1.35.6
```

### Step 3: Check Available Versions in Repos

```bash
# List all available kubeadm versions
sudo dnf list kubeadm --available --showduplicates 2>/dev/null | tail -20

# Or search for specific versions
sudo dnf search kubeadm | grep -i "1.3"
```

### Step 4: Add Official Kubernetes Repository (If Needed)

Rocky Linux repos may not have the latest versions. Add the official Kubernetes repository:

```bash
# Add Kubernetes official repo
sudo tee /etc/yum.repos.d/kubernetes.repo > /dev/null <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
EOF

# Update DNF cache
sudo dnf makecache

# Verify repo is available
sudo dnf repolist | grep kubernetes
```

### Step 5: Verify Available Package Versions

```bash
# After adding repo, check available versions
sudo dnf list kubeadm --available --showduplicates | grep kubeadm

# Should now show:
# kubeadm.x86_64    1.30.14-0    @baseos
# kubeadm.x86_64    1.31.x-0     kubernetes
# kubeadm.x86_64    1.32.x-0     kubernetes
# ... up to 1.35.6
```

### Step 6: Plan Your Upgrade Strategy

**If upgrading from v1.30.14 to v1.35.6:**

This workshop will focus on **v1.30 → v1.31** first, as an example upgrade cycle.

Repeat the same process for each minor version upgrade:

```
Week 1: 1.30.14 → 1.31.x (follow this guide)
Week 2: 1.31.x → 1.32.x (repeat guide with version numbers)
Week 3: 1.32.x → 1.33.x (repeat guide with version numbers)
Week 4: 1.33.x → 1.34.x (repeat guide with version numbers)
Week 5: 1.34.x → 1.35.6 (repeat guide with version numbers)
```

**Total upgrade timeline: 4-5 weeks (one minor version per week)**

### Step 7: Upgrade kubeadm, kubelet, and kubectl to Next Minor Version

⚠️ **IMPORTANT**: Always upgrade all three packages (kubeadm, kubelet, kubectl) to the SAME version

```bash
# Find the next available minor version
# Example: upgrading from 1.30.14 to 1.31.x

# Check what's available
sudo dnf list kubeadm-1.31.* --available

# Option A: Upgrade to the latest 1.31.x (RECOMMENDED)
sudo dnf upgrade kubeadm-1.31.* kubelet-1.31.* kubectl-1.31.* -y

# Option B: Upgrade to specific version (for consistency)
sudo dnf upgrade kubeadm-1.31.5 kubelet-1.31.5 kubectl-1.31.5 -y

# Verify ALL three packages are upgraded to same version
kubeadm version
kubelet --version
kubectl version --short

# Example output (all should match):
# kubeadm version: v1.31.5
# Kubelet v1.31.5
# Client Version: v1.31.5
```

### Important: Version Parity

```bash
# MUST HAVE: All three versions matching
✓ kubeadm v1.31.5
✓ kubelet v1.31.5
✓ kubectl v1.31.5

# DO NOT ALLOW: Version mismatches
✗ kubeadm v1.31.5 + kubelet v1.30.14 (WILL FAIL)
✗ kubectl v1.31.0 + kubelet v1.31.5 (MAY FAIL)

# Verify version parity
echo "=== Version Check ===" && \
echo "kubeadm: $(kubeadm version -o short)" && \
echo "kubelet: $(kubelet --version)" && \
echo "kubectl: $(kubectl version --short 2>/dev/null | grep -i client)"
```

### What If Versions Don't Match?

```bash
# Force install exact matching versions
sudo dnf reinstall kubeadm-1.31.5 kubelet-1.31.5 kubectl-1.31.5 -y

# Or clean and reinstall
sudo dnf remove kubeadm kubelet kubectl -y
sudo dnf install kubeadm-1.31.5 kubelet-1.31.5 kubectl-1.31.5 -y

# Verify again
kubeadm version
kubelet --version
kubectl version --short
```

### Troubleshooting Version Lookup

```bash
# If dnf can't find the package:

# 1. Clear cache
sudo dnf clean all

# 2. Rebuild cache
sudo dnf makecache

# 3. Check repo connectivity
sudo dnf repolist -v

# 4. Try generic version (uses latest in repo)
sudo dnf upgrade kubeadm kubelet kubectl -y
```

---

## Pre-Upgrade Checklist

### 1. Verify Current Cluster State and Version Parity

```bash
# Check current versions of all three components
echo "=== Kubernetes Component Versions ===" && \
kubeadm version -o short && \
kubelet --version && \
kubectl version --short 2>/dev/null | grep Client

# Verify all three versions MATCH
# Example (all should show v1.30.14):
# v1.30.14
# Kubelet v1.30.14
# Client Version: v1.30.14

# Check cluster nodes
kubectl get nodes -o wide

# Check cluster health
kubectl get componentstatuses
kubectl cluster-info

# Verify no version mismatches (critical!)
# If versions don't match, fix before proceeding:
# sudo dnf remove kubeadm kubelet kubectl -y
# sudo dnf install kubeadm-1.30.14 kubelet-1.30.14 kubectl-1.30.14 -y
```

### 2. Backup Critical Data
```bash
# Backup etcd (on control plane)
sudo mkdir -p /backup
sudo cp -r /var/lib/etcd /backup/etcd-backup-$(date +%Y%m%d-%H%M%S)

# Backup kubeadm configuration
sudo cp /etc/kubernetes/admin.conf /backup/admin.conf.bak
sudo cp /etc/kubernetes/manifests /backup/manifests-bak -r

# Backup Flannel configuration
kubectl get daemonset -n kube-flannel -o yaml > /backup/flannel-daemonset.yaml
kubectl get cm -n kube-flannel -o yaml > /backup/flannel-configmap.yaml
```

### 3. Check Available Kubernetes Versions
```bash
# List available kubeadm versions in dnf
sudo dnf list kubeadm --available | grep 1.35

# Should show versions like: kubeadm.x86_64    1.35.6-0
```

### 4. Review Upgrade Path
```
v1.30.14 → v1.31.x → v1.32.x → v1.33.x → v1.34.x → v1.35.6

Note: MUST upgrade sequentially through each minor version
      Cannot skip minor versions (e.g., 1.30 → 1.35 will FAIL)
      
This guide covers ONE upgrade cycle (e.g., 1.30 → 1.31)
Repeat the same steps for each subsequent version
```

### 5. Pre-flight Checks
```bash
# Verify all nodes are Ready
kubectl get nodes
# All should show "Ready" status

# Check persistent volumes
kubectl get pv
kubectl get pvc

# Verify no pending pod evictions
kubectl get pods -A | grep -i evict

# Check for deprecated APIs
kubectl api-resources
```

---

## Preparation & Planning

### 1. Create Maintenance Window
- Schedule during low-traffic period (2-4 hours minimum)
- Notify team members
- Have rollback plan ready

### 2. Understand Your Environment
```bash
# Check control plane nodes
kubectl get nodes --selector node-role.kubernetes.io/control-plane

# Check worker nodes
kubectl get nodes --selector '!node-role.kubernetes.io/control-plane'

# Check pod distribution
kubectl get pods -A -o wide

# Check for local storage
kubectl get pods -A -o json | jq '.items[] | select(.spec.volumes[]?.hostPath) | .metadata.name'
```

### 3. Drain & Cordon Strategy
```bash
# For each node we'll:
# 1. Cordon it (prevent new pods)
# 2. Drain it (remove existing pods)
# 3. Upgrade kubeadm, kubelet, kubectl
# 4. Uncordon it (allow new pods)
```

### 4. Network Verification (Flannel-specific)
```bash
# Verify Flannel is healthy
kubectl get pods -n kube-flannel
kubectl logs -n kube-flannel -l app=flannel --tail=20

# Verify Flannel network
ip link show | grep flannel
ip route | grep flannel

# Check Flannel backend
kubectl get cm kube-flannel-cfg -n kube-flannel -o jsonpath='{.data.net-conf\.json}' | jq .
```

---

## Control Plane Node Upgrade

### ⚠️ CRITICAL: Upgrade Control Plane FIRST

### Step 1: Drain Control Plane Node
```bash
# Get control plane node name
CONTROL_PLANE=$(kubectl get nodes --selector node-role.kubernetes.io/control-plane -o jsonpath='{.items[0].metadata.name}')

echo "Draining control plane: $CONTROL_PLANE"

# Drain the node (evict all pods)
kubectl drain $CONTROL_PLANE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --skip-wait-for-delete-timeout=30s

# Verify node is drained
kubectl get pods -A -o wide | grep $CONTROL_PLANE
# Should show only daemonsets (like flannel, kube-proxy)
```

### Step 2: SSH to Control Plane Node
```bash
ssh user@<control-plane-ip>

# Switch to root
sudo su -

# Verify you're on correct node
hostname
```

### Step 3: Stop kubelet Before Upgrade
```bash
sudo systemctl stop kubelet
sudo systemctl status kubelet
```

### Step 4: Upgrade kubeadm, kubelet, and kubectl Together

⚠️ **CRITICAL**: Upgrade all three to the SAME version simultaneously

```bash
# Update DNF cache
sudo dnf update -y

# Upgrade all three components to same version
# Example: upgrading to v1.31.5
sudo dnf upgrade kubeadm-1.31.5 kubelet-1.31.5 kubectl-1.31.5 -y

# Or upgrade to latest in current minor version
sudo dnf upgrade kubeadm-1.31.* kubelet-1.31.* kubectl-1.31.* -y

# Verify ALL three are now at same version
echo "=== Verifying Version Parity ===" && \
kubeadm version -o short && \
kubelet --version && \
kubectl version --short 2>/dev/null | grep Client

# All three should show same version (e.g., v1.31.5)
```

### Step 5: Plan Upgrade
```bash
# Check upgrade plan
sudo kubeadm upgrade plan

# Output will show:
# - Current version
# - Available versions
# - Required upgrades
```

### Step 6: Apply Upgrade
```bash
# Apply the actual upgrade
sudo kubeadm upgrade apply v1.35.6 -y

# Wait for output like:
# [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.35.6"
```

### Step 7: Verify kubelet and kubectl Were Upgraded

```bash
# kubelet and kubectl should already be at new version (upgraded in Step 4)
# Just verify they match kubeadm

echo "=== Confirming All Three Packages Match ===" && \
echo "kubeadm: $(kubeadm version -o short)" && \
echo "kubelet: $(kubelet --version | awk '{print $2}')" && \
echo "kubectl: $(kubectl version --short 2>/dev/null | grep Client | awk '{print $3}')"

# All should show same version
# If NOT matching, reinstall:
DESIRED_VERSION="1.31.5"  # Change to your target version
sudo dnf reinstall kubeadm-${DESIRED_VERSION} kubelet-${DESIRED_VERSION} kubectl-${DESIRED_VERSION} -y
```

### Step 8: Restart kubelet
```bash
# Daemon reload for systemd
sudo systemctl daemon-reload

# Start kubelet
sudo systemctl start kubelet
sudo systemctl status kubelet

# Verify it's running
sudo systemctl is-active kubelet
# Output: active
```

### Step 9: Monitor Control Plane Recovery
```bash
# Back on your laptop/management machine:
export KUBECONFIG=/path/to/admin.conf

# Wait for control plane to be ready (5-10 minutes)
kubectl get nodes
# Wait until control-plane shows "Ready"

# Check component health
kubectl get componentstatuses

# Watch control plane pods
kubectl get pods -n kube-system -w
# Wait for all to be Running/Ready
```

### Step 10: Uncordon Control Plane
```bash
kubectl uncordon $CONTROL_PLANE

# Verify
kubectl get nodes
# Control plane should show "Ready" without "SchedulingDisabled"
```

---

## Worker Node Upgrades

### ⚠️ Upgrade ONE worker node at a time

### For Each Worker Node:

#### Step 1: Identify Worker Nodes
```bash
# List all worker nodes
kubectl get nodes --selector '!node-role.kubernetes.io/control-plane' -o wide
```

#### Step 2: Drain Worker Node
```bash
# Set worker node name
WORKER_NODE="node-2"  # Change to actual node name

echo "Draining worker node: $WORKER_NODE"

# Drain pods gracefully
kubectl drain $WORKER_NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --skip-wait-for-delete-timeout=30s \
  --timeout=5m

# Verify node is drained
kubectl get pods -A -o wide | grep $WORKER_NODE
# Should only show daemonsets
```

#### Step 3: Cordon Worker Node (Safety Check)
```bash
# Prevent new pods from being scheduled
kubectl cordon $WORKER_NODE

# Verify
kubectl get nodes | grep $WORKER_NODE
# Should show "SchedulingDisabled"
```

#### Step 4: SSH to Worker Node
```bash
ssh user@<worker-node-ip>
sudo su -
```

#### Step 5: Stop kubelet
```bash
sudo systemctl stop kubelet
sudo systemctl status kubelet
```

#### Step 6: Upgrade kubeadm, kubelet, and kubectl Together

```bash
# Update DNF cache
sudo dnf update -y

# Upgrade all three to SAME version
# Example: upgrading to v1.31.5
sudo dnf upgrade kubeadm-1.31.5 kubelet-1.31.5 kubectl-1.31.5 -y

# Or upgrade to latest in minor version
sudo dnf upgrade kubeadm-1.31.* kubelet-1.31.* kubectl-1.31.* -y

# Verify all three are now at same version
echo "=== Verifying Version Parity ===" && \
kubeadm version -o short && \
kubelet --version && \
kubectl version --short 2>/dev/null | grep Client
```

#### Step 7: Run kubeadm upgrade node

```bash
# On worker nodes, use:
sudo kubeadm upgrade node

# Output: [upgrade] Successfully upgraded kubelet
```

#### Step 8: Verify All Three Packages Match

```bash
# Confirm kubeadm, kubelet, and kubectl all match
echo "=== Version Parity Check ===" && \
echo "kubeadm: $(kubeadm version -o short)" && \
echo "kubelet: $(kubelet --version | awk '{print $2}')" && \
echo "kubectl: $(kubectl version --short 2>/dev/null | grep Client | awk '{print $3}')"

# If mismatch, reinstall:
DESIRED_VERSION="1.31.5"  # Change to your target version
sudo dnf reinstall kubeadm-${DESIRED_VERSION} kubelet-${DESIRED_VERSION} kubectl-${DESIRED_VERSION} -y
```

#### Step 9: Restart kubelet
```bash
sudo systemctl daemon-reload
sudo systemctl start kubelet
sudo systemctl status kubelet
```

#### Step 10: Back to Management Machine - Uncordon Node
```bash
# From your management machine
kubectl uncordon $WORKER_NODE

# Verify node is ready
kubectl get nodes | grep $WORKER_NODE
# Should show "Ready" without "SchedulingDisabled"

# Watch for pods to reschedule
kubectl get pods -A -o wide -w | grep $WORKER_NODE
```

#### Step 11: Verify Node Health Before Next Worker
```bash
# Wait 2-3 minutes for pods to stabilize
kubectl get nodes
kubectl get pods -n kube-system

# All pods should be Running/Ready before upgrading next worker
```

#### Repeat Steps 1-11 for Each Remaining Worker Node

---

## Verification & Testing

### 1. Full Cluster Status Check
```bash
# All nodes should be Ready
kubectl get nodes -o wide

# All control plane components should be Running
kubectl get pods -n kube-system -o wide

# Check component statuses
kubectl get componentstatuses

# Verify API server is responsive
kubectl cluster-info
```

### 2. Kubernetes Version Verification
```bash
# Check cluster version
kubectl version --short
# Should show: v1.35.6

# Check each node's kubelet version
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
# All should show: v1.35.6
```

### 3. Flannel Network Verification
```bash
# Verify Flannel pods are running
kubectl get pods -n kube-flannel -o wide
# All should show Running

# Check Flannel logs for errors
kubectl logs -n kube-flannel -l app=flannel --all-containers=true | grep -i error | head -20

# Verify Flannel interfaces on nodes
# SSH to each node:
ip link show | grep flannel
ip route | grep flannel
```

### 4. Pod Communication Test
```bash
# Create test pods on different nodes
kubectl run test-pod-1 --image=alpine --restart=Never -- sleep 3600
kubectl run test-pod-2 --image=alpine --restart=Never -- sleep 3600

# Get pod IPs and nodes
kubectl get pods -o wide

# Test connectivity between pods
POD1_IP=$(kubectl get pod test-pod-1 -o jsonpath='{.status.podIP}')
POD2_IP=$(kubectl get pod test-pod-2 -o jsonpath='{.status.podIP}')

kubectl exec test-pod-1 -- ping -c 3 $POD2_IP
kubectl exec test-pod-2 -- ping -c 3 $POD1_IP

# Should show successful pings

# Cleanup
kubectl delete pod test-pod-1 test-pod-2
```

### 5. Service Connectivity Test
```bash
# Test service DNS
kubectl run test-dns --image=alpine --restart=Never -- nslookup kubernetes.default

# Test service access
kubectl get svc kubernetes
kubectl get endpoints kubernetes

# Verify API server is accessible
kubectl get services kubernetes
kubectl describe svc kubernetes
```

### 6. Workload Stability Test
```bash
# Verify all workloads are healthy
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get daemonsets -A

# Check for any unhealthy pods
kubectl get pods -A --field-selector=status.phase!=Running
# Should return empty (no unhealthy pods)

# Check pod events for errors
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```

### 7. Storage & PersistentVolume Test (if applicable)
```bash
# Verify persistent volumes
kubectl get pv
kubectl get pvc -A

# Check volume mounting
kubectl get pods -A -o json | jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim) | {name: .metadata.name, pvc: .spec.volumes[].persistentVolumeClaim.claimName}'
```

---

## Troubleshooting

### Issue 1: kubelet Won't Start After Upgrade
```bash
# Check kubelet logs
sudo journalctl -u kubelet -n 50

# Common issues:
# 1. Cgroup driver mismatch
sudo kubeadm config view
# Check cgroupDriver setting

# 2. Kubelet configuration
sudo cat /var/lib/kubelet/kubeadm-flags.env

# 3. Reset and rejoin
sudo kubeadm reset --force
sudo kubeadm join <cluster-info> --token=<token> --discovery-token-ca-cert-hash=sha256:<hash>
```

### Issue 2: Pods Not Scheduling After Upgrade
```bash
# Check node readiness
kubectl get nodes
# If NotReady, check kubelet

# Check kubelet status on node
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100

# Restart kubelet
sudo systemctl restart kubelet

# Wait 2-3 minutes
sleep 180
kubectl get nodes
```

### Issue 3: DNS Not Working (Flannel-specific)
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system | grep coredns

# If not running:
kubectl rollout restart deployment/coredns -n kube-system

# Check Flannel
kubectl get pods -n kube-flannel

# Restart Flannel if needed
kubectl rollout restart daemonset/kube-flannel-ds -n kube-flannel

# Test DNS
kubectl run -it --rm debug --image=alpine --restart=Never -- nslookup kubernetes.default
```

### Issue 4: Certificate Errors During Upgrade
```bash
# Check certificate expiry
sudo kubeadm certs check-expiration

# Renew certificates
sudo kubeadm certs renew all

# Restart control plane components
sudo systemctl restart kubelet
sleep 30
kubectl get componentstatuses
```

### Issue 5: Node Status Stuck NotReady
```bash
# SSH to node
ssh user@<node-ip>

# Check disk space (v1.35 requires more space)
df -h

# Check system resources
free -h
top -b -n 1 | head -20

# If low on resources:
# 1. Clear old container logs
sudo journalctl --vacuum=100M

# 2. Clean up Docker/containerd
sudo docker system prune -a
# or
sudo crictl rmi --prune

# 3. Restart kubelet
sudo systemctl restart kubelet
```

### Issue 6: etcd Issues on Control Plane
```bash
# Check etcd status
sudo systemctl status etcd

# View etcd logs
sudo journalctl -u etcd -n 100

# Check etcd health (from control plane)
sudo ETCDCTL_API=3 etcdctl --endpoints=127.0.0.1:2379 endpoint health

# If corrupted, restore from backup:
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd/*
sudo cp -r /backup/etcd-backup-<date>/* /var/lib/etcd/
sudo chown -R etcd:etcd /var/lib/etcd
sudo systemctl start etcd
```

### Issue 7: Version Mismatch Between kubeadm, kubelet, and kubectl

**Symptoms**: kubeadm works but kubelet fails to start, or kubectl can't communicate

```bash
# Check all three versions
echo "=== Version Check ===" && \
kubeadm version -o short && \
kubelet --version && \
kubectl version --short 2>/dev/null | grep Client

# Example of WRONG output (versions don't match):
# v1.31.5
# Kubelet v1.30.14    ← MISMATCH!
# Client Version: v1.31.5    ← MISMATCH!

# Fix: Reinstall all three to same version
TARGET_VERSION="1.31.5"
sudo systemctl stop kubelet
sudo dnf remove kubeadm kubelet kubectl -y
sudo dnf install kubeadm-${TARGET_VERSION} kubelet-${TARGET_VERSION} kubectl-${TARGET_VERSION} -y

# Verify parity
kubeadm version -o short
kubelet --version
kubectl version --short 2>/dev/null | grep Client

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl start kubelet
```

### Issue 8: Flannel MTU/Network Fragmentation
```bash
# Check Flannel MTU settings
kubectl get cm kube-flannel-cfg -n kube-flannel -o jsonpath='{.data.net-conf\.json}' | jq .

# If networking is slow, check MTU:
sudo ip link show | grep -i mtu

# Adjust if needed (on all nodes):
sudo ip link set dev eth0 mtu 1450

# Update Flannel config:
kubectl patch cm kube-flannel-cfg -n kube-flannel --type merge -p '{"data":{"net-conf.json":"{\"Network\":\"10.244.0.0/16\",\"Backend\":{\"Type\":\"vxlan\",\"VNI\":1,\"Port\":8472},\"MTU\":1450}"}}'
```

---

## Rollback Procedures

### ⚠️ Only if Cluster is Unstable

### Full Rollback to v1.3 (Last Resort)

```bash
# 1. Stop all services
kubectl scale deployment --all --replicas=0 -A

# 2. On each node, downgrade packages
sudo dnf downgrade kubeadm-1.3.* kubelet-1.3.* kubectl-1.3.* -y

# 3. Restore kubeadm config
sudo cp /backup/admin.conf /etc/kubernetes/admin.conf

# 4. Restore etcd (on control plane)
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd/*
sudo cp -r /backup/etcd-backup-<date>/* /var/lib/etcd/
sudo chown -R etcd:etcd /var/lib/etcd
sudo systemctl start etcd

# 5. Restart kubelet on all nodes
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. Verify cluster
kubectl get nodes
kubectl cluster-info
```

### Selective Rollback (Single Node)

```bash
# If only one worker node has issues:

WORKER_NODE="node-2"

# 1. Cordon node
kubectl cordon $WORKER_NODE

# 2. Drain node
kubectl drain $WORKER_NODE --ignore-daemonsets --delete-emptydir-data

# 3. SSH to node
ssh user@<node-ip>

# 4. Downgrade
sudo dnf downgrade kubeadm-1.30.* kubelet-1.30.* kubectl-1.30.* -y

# 5. Restart services
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. Uncordon from management machine
kubectl uncordon $WORKER_NODE
```

---

## Post-Upgrade Checklist

- [ ] All nodes show "Ready" status
- [ ] All control plane pods are Running
- [ ] Flannel network is functional (no red logs)
- [ ] Test pod can reach DNS
- [ ] Service-to-service communication works
- [ ] Persistent volumes mount correctly
- [ ] No pending pod evictions
- [ ] All workloads scheduled and healthy
- [ ] etcd is healthy
- [ ] Certificate expiry is > 100 days
- [ ] Backup cluster configuration:
  ```bash
  kubectl get all -A -o yaml > /backup/cluster-state-v1.35.6.yaml
  ```

---

## Quick Reference Commands

```bash
# Get current cluster version
kubectl version --short

# Check version parity (all three must match)
echo "kubeadm: $(kubeadm version -o short)" && \
echo "kubelet: $(kubelet --version | awk '{print $2}')" && \
echo "kubectl: $(kubectl version --short 2>/dev/null | grep Client | awk '{print $3}')"

# If versions don't match, reinstall all three together
sudo dnf reinstall kubeadm-1.31.5 kubelet-1.31.5 kubectl-1.31.5 -y

# Check node versions
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion

# View all pod statuses
kubectl get pods -A --no-headers | awk '{print $4}' | sort | uniq -c

# Check resource usage
kubectl top nodes
kubectl top pods -A

# View recent events
kubectl get events -A --sort-by='.lastTimestamp'

# Monitor upgrade progress
watch -n 5 'kubectl get nodes && echo "---" && kubectl get pods -n kube-system'

# Drain node for upgrade
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon node after upgrade
kubectl uncordon <node-name>

# Check certificate expiry
sudo kubeadm certs check-expiration

# View kubelet logs
sudo journalctl -u kubelet -f

# Check Flannel status
kubectl get ds -n kube-flannel
kubectl logs -n kube-flannel -l app=flannel -f
```

---

## Notes for v1.30 → v1.35 Upgrade

### Major Changes Between Versions:
- **v1.31**: Enhanced storage drivers, improved scheduler
- **v1.32**: Pod security updates, new metrics
- **v1.33**: Deprecated APIs removed from v1.26+
- **v1.34**: Networking improvements, new admission controllers
- **v1.35**: Latest stable, performance optimizations

### Minor Changes to Expect:
- **API evolutions**: Gradual deprecations (check release notes for each version)
- **CRI updates**: Container runtime interface enhancements
- **Network policies**: Improved egress rules
- **Metrics**: New prometheus metrics added in some versions
- **Storage**: Incremental improvements

### Resource Requirements:
- **Disk**: Increased from 2GB to 5GB+ per node
- **Memory**: Increased from 1GB to 2GB+ minimum
- **CPU**: Recommend 2 cores per node minimum

### Testing Recommendations:
1. Upgrade test cluster first
2. Run workloads for 24 hours
3. Monitor metrics and logs
4. Validate storage and networking
5. Only then upgrade production

---

## Repeating the Upgrade for Subsequent Versions

### After Completing v1.30 → v1.31

Once you've successfully upgraded to v1.31.x and verified cluster health (24+ hours):

```bash
# 1. Wait 1-2 weeks before next upgrade
# 2. Follow EXACT same steps in this guide
# 3. But change version numbers:

# Old (v1.30 → v1.31):
sudo dnf upgrade kubeadm-1.31.* kubelet-1.31.* kubectl-1.31.* -y

# New (v1.31 → v1.32):
sudo dnf upgrade kubeadm-1.32.* kubelet-1.32.* kubectl-1.32.* -y

# Verify
kubeadm version  # Should show v1.32.x
```

### Upgrade Cycle Timeline

| Week | Upgrade | Duration | Testing |
|------|---------|----------|---------|
| 1 | 1.30 → 1.31 | 3-4 hours | 24 hours |
| 2-3 | Testing & Stability | - | - |
| 4 | 1.31 → 1.32 | 3-4 hours | 24 hours |
| 5-6 | Testing & Stability | - | - |
| 7 | 1.32 → 1.33 | 3-4 hours | 24 hours |
| 8-9 | Testing & Stability | - | - |
| 10 | 1.33 → 1.34 | 3-4 hours | 24 hours |
| 11-12 | Testing & Stability | - | - |
| 13 | 1.34 → 1.35 | 3-4 hours | 24 hours |
| 14+ | Final Verification | - | Ongoing |

**Total Timeline**: 14+ weeks (3.5 months)

### Key Points for Each Upgrade Cycle

```bash
# For EACH cycle (1.30→1.31, 1.31→1.32, etc.):

# 1. Update documentation with new version
# 2. Back up etcd
sudo cp -r /var/lib/etcd /backup/etcd-backup-v1.XX

# 3. Follow the exact same procedure from this guide
# 4. Change only the version numbers

# 5. Verify EACH time
kubeadm version
kubectl get nodes
kubectl get pods -A --no-headers | grep -v Running | grep -v Completed

# 6. Monitor for 24 hours
# 7. Only then proceed to next version
```

### Checklist for Each Version Upgrade

- [ ] Backup etcd with new version number
- [ ] Backup admin.conf
- [ ] Drain control plane
- [ ] Upgrade kubeadm on control plane
- [ ] Run `kubeadm upgrade plan`
- [ ] Run `kubeadm upgrade apply vX.XX.X`
- [ ] Upgrade kubelet and kubectl on control plane
- [ ] Restart kubelet
- [ ] Uncordon control plane
- [ ] Wait 5 minutes for stability
- [ ] Drain first worker node
- [ ] Upgrade worker node
- [ ] Uncordon worker node
- [ ] Wait for pods to reschedule (2-3 min)
- [ ] Repeat for each worker node
- [ ] Run full verification suite
- [ ] Monitor metrics for 24 hours
- [ ] Document completion in upgrade log

---

## Support & Documentation

- [Kubernetes Upgrade Documentation](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [kubeadm Reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [Flannel Documentation](https://github.com/coreos/flannel)
- [Kubernetes Release Notes](https://github.com/kubernetes/kubernetes/releases)

---

**Last Updated**: August 2026  
**Kubernetes Version**: 1.35.6  
**Flannel Backend**: VXLAN (default)  
**kubeadm Status**: Stable

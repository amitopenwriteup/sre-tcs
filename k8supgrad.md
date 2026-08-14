# Kubernetes Reset & Fresh Install: v1.35.6
## Guide using kubeadm on Rocky Linux 9 with Flannel

> **Note:** This guide tears the cluster down with `kubeadm reset` and performs a fresh install of v1.35.6. It does not upgrade in place, and it includes no backup steps — all workloads, etcd data, and cluster state are wiped and not recoverable.

### Current Cluster Status
- **Current Version**: v1.30.14
- **Target Version**: v1.35.6
- **Method**: kubeadm reset → fresh install
- **Networking**: Flannel (VXLAN backend)
- **Operating System**: Rocky Linux 9

---

## Table of Contents
1. [Reset the Cluster](#reset-the-cluster)
2. [Install Kubernetes v1.35.6](#install-kubernetes-v1356)
3. [Initialize the New Control Plane](#initialize-the-new-control-plane)
4. [Install Flannel](#install-flannel)
5. [Join Worker Nodes](#join-worker-nodes)
6. [Verification & Testing](#verification--testing)
7. [Troubleshooting](#troubleshooting)

---

## Reset the Cluster

### Step 1: Drain and Remove Worker Nodes

```bash
# From the management machine, for each worker node:
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data --force
kubectl delete node <worker-node>
```

### Step 2: Reset Each Worker Node

```bash
# SSH to each worker node
ssh user@<worker-node-ip>
sudo su -

# Stop kubelet
sudo systemctl stop kubelet

# Reset kubeadm state
sudo kubeadm reset -f

# Clean up CNI and iptables
sudo rm -rf /etc/cni/net.d
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Clean up kubeconfig
rm -rf $HOME/.kube
```

### Step 3: Reset the Control Plane Node

```bash
# SSH to control plane node
ssh user@<control-plane-ip>
sudo su -

# Stop kubelet
sudo systemctl stop kubelet

# Reset kubeadm state (this also wipes etcd data on this node)
sudo kubeadm reset -f

# Clean up CNI and iptables
sudo rm -rf /etc/cni/net.d
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Remove old etcd data directory explicitly
sudo rm -rf /var/lib/etcd

# Clean up kubeconfig
rm -rf $HOME/.kube
```

---

## Install Kubernetes v1.35.6

### Step 1: Remove Old Packages (All Nodes)

```bash
sudo dnf remove kubeadm kubelet kubectl -y
```

### Step 2: Add the Kubernetes v1.35 Repository (All Nodes)

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo > /dev/null <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
EOF

sudo dnf makecache
sudo dnf repolist | grep kubernetes
```

### Step 3: Install kubeadm, kubelet, kubectl v1.35.6 (All Nodes)

```bash
# Confirm the version is available
sudo dnf list kubeadm-1.35.6 --available

# Install all three at the exact same version
sudo dnf install kubeadm-1.35.6 kubelet-1.35.6 kubectl-1.35.6 -y

# Verify
kubeadm version
kubelet --version
kubectl version --short

# All three should show v1.35.6
```

### Step 4: Enable kubelet (All Nodes)

```bash
sudo systemctl enable kubelet
```

---

## Initialize the New Control Plane

### Step 1: Run kubeadm init

```bash
# On the control plane node only
sudo kubeadm init \
  --kubernetes-version=v1.35.6 \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs

# Save the output — it includes the `kubeadm join` command for workers
```

### Step 2: Configure kubectl Access

```bash
# On the control plane node
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
kubectl cluster-info
```

---

## Install Flannel

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Verify Flannel pods come up
kubectl get pods -n kube-flannel -w
```

---

## Join Worker Nodes

```bash
# On each worker node, run the join command saved from `kubeadm init`
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# If the token has expired, generate a new one from the control plane:
kubeadm token create --print-join-command
```

Verify from the management machine:

```bash
kubectl get nodes -o wide
# All nodes should show Ready once Flannel finishes initializing on each
```

---

## Verification & Testing

### 1. Cluster Status
```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl cluster-info
```

### 2. Version Check
```bash
kubectl version --short
# Should show v1.35.6

kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
# All nodes should show v1.35.6
```

### 3. Flannel Network Check
```bash
kubectl get pods -n kube-flannel -o wide
kubectl logs -n kube-flannel -l app=flannel --tail=20
```

### 4. Pod Communication Test
```bash
kubectl run test-pod-1 --image=alpine --restart=Never -- sleep 3600
kubectl run test-pod-2 --image=alpine --restart=Never -- sleep 3600

POD1_IP=$(kubectl get pod test-pod-1 -o jsonpath='{.status.podIP}')
POD2_IP=$(kubectl get pod test-pod-2 -o jsonpath='{.status.podIP}')

kubectl exec test-pod-1 -- ping -c 3 $POD2_IP
kubectl exec test-pod-2 -- ping -c 3 $POD1_IP

kubectl delete pod test-pod-1 test-pod-2
```

### 5. DNS Test
```bash
kubectl run -it --rm debug --image=alpine --restart=Never -- nslookup kubernetes.default
```

---

## Troubleshooting

### Issue 1: kubeadm init Fails
```bash
# Check for leftover state from the reset
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/etcd $HOME/.kube

# Retry init
sudo kubeadm init --kubernetes-version=v1.35.6 --pod-network-cidr=10.244.0.0/16 --upload-certs
```

### Issue 2: Worker Node Won't Join
```bash
# Confirm worker was fully reset
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube

# Generate a fresh join command from the control plane
kubeadm token create --print-join-command

# Retry join on the worker
```

### Issue 3: kubelet Won't Start
```bash
sudo journalctl -u kubelet -n 50

# Check cgroup driver matches containerd config
sudo kubeadm config view
cat /etc/containerd/config.toml | grep -i cgroup

sudo systemctl restart kubelet
```

### Issue 4: Flannel Pods CrashLoop
```bash
kubectl logs -n kube-flannel -l app=flannel --tail=50

# Confirm pod-network-cidr matches Flannel's expected 10.244.0.0/16
kubectl get cm kube-flannel-cfg -n kube-flannel -o jsonpath='{.data.net-conf\.json}' | jq .

kubectl rollout restart daemonset/kube-flannel-ds -n kube-flannel
```

### Issue 5: Node Stuck NotReady
```bash
ssh user@<node-ip>
df -h
free -h
sudo systemctl restart kubelet
sudo journalctl -u kubelet -n 100
```

---

## Post-Install Checklist

- [ ] All nodes show "Ready"
- [ ] All control plane pods Running
- [ ] Flannel pods Running, no error logs
- [ ] DNS resolves inside cluster
- [ ] Pod-to-pod communication works across nodes
- [ ] Certificates valid (`kubeadm certs check-expiration`)

---

## Quick Reference Commands

```bash
# Reset a node
sudo kubeadm reset -f

# Install exact version
sudo dnf install kubeadm-1.35.6 kubelet-1.35.6 kubectl-1.35.6 -y

# Init control plane
sudo kubeadm init --kubernetes-version=v1.35.6 --pod-network-cidr=10.244.0.0/16 --upload-certs

# Get a fresh join command
kubeadm token create --print-join-command

# Check versions
kubeadm version -o short
kubelet --version
kubectl version --short

# Check node status
kubectl get nodes -o wide

# Check Flannel
kubectl get pods -n kube-flannel
kubectl logs -n kube-flannel -l app=flannel -f
```

---

## Support & Documentation

- [kubeadm reset Reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
- [kubeadm init Reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
- [Flannel Documentation](https://github.com/flannel-io/flannel)
- [Kubernetes Release Notes](https://github.com/kubernetes/kubernetes/releases)

---

**Last Updated**: August 2026
**Kubernetes Version**: 1.35.6
**Flannel Backend**: VXLAN (default)
**Method**: kubeadm reset + fresh install

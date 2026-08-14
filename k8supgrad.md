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



### Step 2: Reset Each Worker Node

```bash
# SSH to each worker node

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


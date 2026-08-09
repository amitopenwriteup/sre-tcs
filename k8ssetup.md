# Setup Kubernetes Master using kubeadm — Rocky Linux

**Applies to:** Rocky Linux 8 / 9
**Container runtime:** containerd (via Docker's repo)
**Kubernetes version:** v1.30

We will be using containerd as the container runtime. Docker Engine's repo is used only to source the `containerd.io` package — Kubernetes itself talks to containerd directly, not to the Docker daemon.

Use `sudo` or log in as root: `sudo su -`

---

## Lab: Repo Configuration and Installation (Rocky Linux)

Run in **both** the master and worker VMs (Step 1, Step 2, and Step 3).

### Step 1 — Kernel modules and sysctl params

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

### Step 2 — Install containerd, kubelet, kubeadm, kubectl

Install prerequisites:

```bash
sudo dnf install -y dnf-plugins-core curl gnupg2
```

Add the Docker CE repo (source of the `containerd.io` package) and install containerd:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y containerd.io
```

Add the Kubernetes repo:

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

Install kubelet, kubeadm, and kubectl, and lock their versions so a routine `dnf upgrade` doesn't silently break the cluster:

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo dnf versionlock add kubelet kubeadm kubectl
```

### Step 3 — Disable swap

Swap space can potentially interfere with kubelet's resource isolation. When a system is under memory pressure and starts swapping out memory to disk, it can lead to unpredictable performance and behavior for applications running inside containers.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

On Rocky Linux 9 with `zram-generator`-based swap (if present), also disable and mask it:

```bash
sudo systemctl disable --now zram-generator.service 2>/dev/null || true
```

### Step 4 — SELinux and firewalld

Rocky Linux ships with SELinux in enforcing mode and firewalld active, neither of which is present by default on an Ubuntu lab image. Both need to be addressed before `kubeadm init` will succeed.

Set SELinux to permissive (the standard kubeadm-supported approach until you have Kubernetes-aware SELinux policies in place):

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

Open the ports kubeadm and the pod network need. On the **master**:

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10259/tcp
sudo firewall-cmd --permanent --add-port=10257/tcp
sudo firewall-cmd --reload
```

On **worker nodes**:

```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload
```

If you're using Flannel (as this guide does in Step 8), also open its overlay port on every node:

```bash
sudo firewall-cmd --permanent --add-port=8472/udp
sudo firewall-cmd --reload
```

> If this is a throwaway lab environment, `sudo systemctl disable --now firewalld` is a simpler alternative to opening individual ports — not recommended beyond a lab.

### Step 5 — Configure containerd for systemd cgroups

This is important because Kubernetes requires all its components, and the container runtime, to use systemd for cgroups.

```bash
sudo sh -c "containerd config default > /etc/containerd/config.toml"
sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd.service
sudo systemctl restart kubelet.service
sudo systemctl enable kubelet.service
```

---

## Step 6: Runs only on the master node

### Initialize the cluster with kubeadm

`kubeadm init` is used to initialize a Kubernetes control-plane node. The `--pod-network-cidr` flag specifies the range of IP addresses for the pod network in your cluster — `10.244.0.0/16` matches Flannel's default expected range.

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

To start using your cluster, you need to run the following as a regular user.

**Note:** copy the `kubeadm join` output and save it in a notepad — you'll need it on every worker node.

---

## Step 7: Runs only on the master node

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Step 8: Install the network plugin — master node only

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Run the command below after ~5 minutes:

```bash
kubectl get nodes

#Node must come ready
#NAME       STATUS   ROLES           AGE     VERSION
#rkymaster  Ready    control-plane   4m40s   v1.30.x

#If "Not Ready" persists, restart the containerd service
sudo systemctl restart containerd
```

---

## Join Worker Nodes

Now run the `kubeadm join` command on the client (worker) node — repeat Step 1, Step 2, and Step 3 above on the worker first, then run the join command you copied after `kubeadm init`.

If you missed the `kubeadm join` command, regenerate it on the master:

```bash
kubeadm token create --print-join-command
```

On the worker node:

```bash
sudo modprobe br_netfilter
echo '1' | sudo tee /proc/sys/net/ipv4/ip_forward

# Below is an example — run your own copied kubeadm join command
# sudo kubeadm join 172.16.207.130:6443 --token r6i5ud.xyt1242cyo95ig68 \
#   --discovery-token-ca-cert-hash sha256:6717f453c5a347dcec6499b63fad0351d1986d19f2f8c3455dc8b3d03707e16a

sudo mkdir -p /root/.kube
sudo cp /etc/kubernetes/kubelet.conf /root/.kube/config
kubectl get nodes
```

---

## Troubleshooting notes specific to Rocky Linux

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubeadm init` hangs or fails at preflight checks on ports/SELinux | SELinux still enforcing, or firewalld blocking required ports | Re-check Step 4; confirm `getenforce` returns `Permissive` |
| Worker never reaches `Ready` | Flannel overlay port `8472/udp` blocked between nodes | Open it on every node per Step 4 |
| `kubelet` fails to start with cgroup driver mismatch | `SystemdCgroup` not set to `true`, or containerd not restarted after the edit | Re-run Step 5, confirm with `sudo systemctl status containerd` |
| `dnf upgrade` later breaks the cluster | kubelet/kubeadm/kubectl upgraded out of band | Confirm lock is active: `sudo dnf versionlock list` |
| Nodes show `NotReady` after reboot | Swap re-enabled, or SELinux config reverted on reboot | Confirm `swapon -s` is empty and `/etc/selinux/config` still shows `permissive` |

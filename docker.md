# Uninstalling Podman and Installing Docker on Rocky Linux

**Applies to:** Rocky Linux 8 / 9
**Requires:** `sudo` / root access

---

## 1. Uninstall Podman and related packages

```bash
sudo dnf remove -y podman podman-docker podman-compose podman-plugins buildah skopeo
```

> `podman-docker` provides the `docker` CLI shim that redirects to Podman — removing it is important so it doesn't conflict with the real Docker CLI you're about to install.

### 1.1 Clean up leftover config and data directories

```bash
sudo rm -rf /var/lib/containers
sudo rm -rf /etc/containers
rm -rf ~/.local/share/containers
rm -rf ~/.config/containers
```

**Checkpoint:** confirm Podman is gone:
```bash
podman --version
```
This should return `command not found`.

---

## 2. Install Docker

### 2.1 Remove any old/conflicting Docker packages

```bash
sudo dnf remove -y docker \
  docker-client \
  docker-client-latest \
  docker-common \
  docker-latest \
  docker-latest-logrotate \
  docker-logrotate \
  docker-engine
```

### 2.2 Install prerequisite packages

```bash
sudo dnf install -y dnf-plugins-core
```

### 2.3 Add the Docker CE repository

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

> Rocky Linux is RHEL-compatible, so the CentOS repo works correctly here — this is the officially supported path for RHEL-family distros.

### 2.4 Install Docker Engine, CLI, and plugins

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 2.5 Start and enable Docker

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### 2.6 Verify the installation

```bash
sudo docker --version
sudo docker run hello-world
```

You should see Docker's "Hello from Docker!" confirmation message.

---

## 3. Post-install: run Docker without `sudo` (optional but recommended)

```bash
sudo groupadd docker 2>/dev/null
sudo usermod -aG docker $USER
newgrp docker
```

Log out and back in (or reboot) for the group change to fully apply, then confirm:

```bash
docker run hello-world
```

---

## 4. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `docker: command not found` after install | Shell session predates PATH update, or `podman-docker` shim still present | Open a new shell; re-run `sudo dnf remove podman-docker` if `docker` still points to Podman (`which docker`) |
| `Cannot connect to the Docker daemon` | Docker service not running | `sudo systemctl status docker` → `sudo systemctl start docker` |
| `permission denied` on `docker.sock` | User not in `docker` group yet, or session not refreshed | Re-run `newgrp docker` or log out/in |
| SELinux denials in container logs | Default SELinux policies affecting bind mounts | Use `:z` or `:Z` mount flags, or check `sudo ausearch -m avc -ts recent` |
| Old `firewalld` rules blocking container networking | Podman's CNI/netavark rules left behind | `sudo firewall-cmd --reload`; inspect `sudo firewall-cmd --list-all` |

---

## 5. Summary of what changed

- **Removed:** Podman, Buildah, Skopeo, `podman-docker` shim, and all associated config/data
- **Installed:** Docker CE, Docker CLI, containerd.io, Buildx plugin, Compose plugin
- **Enabled:** `docker.service` on boot, non-root Docker usage via the `docker` group

# Resetting and Cleaning Up a Kubernetes Installation

> **Purpose:** Complete teardown of a Kubernetes node — removes all cluster state, packages, configuration, and container runtime. Run this on each node you want to reset. Useful when rebuilding a cluster from scratch, decommissioning a node, or recovering from a broken installation.

## ⚠️ Warning

This procedure is **destructive and irreversible**. It will:

- Remove the node from the cluster
- Delete all pods, containers, and volumes on this node
- Remove etcd data (if run on a master node)
- Uninstall Kubernetes and the container runtime entirely

> **If you only need to remove a worker node from the cluster** (without wiping everything), use `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` followed by `kubectl delete node <node>` on the master, then run this procedure on the worker.

---

## Step 1: Reset kubeadm and Flush Network Rules

```bash
sudo kubeadm reset -f
```

Flush all iptables rules created by kube-proxy and CNI plugins:

```bash
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X
```

If your cluster used IPVS (common with Calico):

```bash
sudo ipvsadm --clear 2>/dev/null || true
```

---

## Step 2: Stop and Disable Services

```bash
sudo systemctl stop kubelet
sudo systemctl disable kubelet

# Stop the container runtime (use whichever is installed)
# Containerd
sudo systemctl stop containerd
sudo systemctl disable containerd

# Docker (if used instead of Containerd)
sudo systemctl stop docker
sudo systemctl disable docker
```

---

## Step 3: Remove Packages

### RHEL / CentOS / Rocky Linux

```bash
# Kubernetes components
sudo dnf remove -y kubeadm kubectl kubelet kubernetes-cni cri-tools

# Container runtime — remove whichever is installed
sudo dnf remove -y containerd.io
sudo dnf remove -y docker-ce docker-ce-cli
```

### Ubuntu / Debian

```bash
# Kubernetes components
sudo apt-get remove -y kubeadm kubectl kubelet kubernetes-cni cri-tools
sudo apt-get autoremove -y

# Container runtime
sudo apt-get remove -y containerd.io
sudo apt-get remove -y docker-ce docker-ce-cli
```

---

## Step 4: Remove All Data Directories

```bash
# Kubernetes state and configuration
sudo rm -rf /etc/kubernetes
sudo rm -rf ~/.kube
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/etcd
sudo rm -rf /var/run/kubernetes
sudo rm -rf /var/lib/dockershim

# Container runtime data
sudo rm -rf /var/lib/containerd
sudo rm -rf /var/lib/docker          # Only if Docker was used

# CNI plugin binaries and config
sudo rm -rf /opt/cni/bin
sudo rm -rf /etc/cni/net.d

# Logs
sudo rm -rf /var/log/pods
sudo rm -rf /var/log/kube*
sudo rm -rf /var/log/containers
```

---

## Step 5: Remove Package Repositories

### RHEL / CentOS / Rocky Linux

```bash
sudo rm -f /etc/yum.repos.d/kubernetes.repo
sudo rm -f /etc/yum.repos.d/docker-ce.repo
sudo dnf clean all
```

### Ubuntu / Debian

```bash
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo rm -f /etc/apt/sources.list.d/docker.list
sudo apt-get update
```

---

## Step 6: Clean Up Kernel Modules and Sysctl (Optional)

Remove the configurations added during the Kubernetes installation:

```bash
# Remove persistent kernel module config
sudo rm -f /etc/modules-load.d/kubernetes.conf

# Remove sysctl overrides
sudo rm -f /etc/sysctl.d/kubernetes.conf
sudo sysctl --system
```

---

## Step 7: Re-enable Swap (Optional)

If you disabled swap for Kubernetes and want to restore it:

```bash
# Uncomment the swap line in /etc/fstab
sudo sed -i '/#.*swap/s/^#//' /etc/fstab

# Enable swap
sudo swapon -a

# Verify
free -h
```

---

## Step 8: Reboot

```bash
sudo reboot
```

A reboot ensures all kernel modules are unloaded, network state is clean, and no orphaned processes remain.

---

## Verification After Reboot

Confirm everything is removed:

```bash
# No Kubernetes binaries
which kubeadm kubectl kubelet 2>/dev/null && echo "WARN: binaries still found" || echo "OK: binaries removed"

# No running services
systemctl is-active kubelet 2>/dev/null || echo "OK: kubelet not running"
systemctl is-active containerd 2>/dev/null || echo "OK: containerd not running"

# No leftover directories
for dir in /etc/kubernetes /var/lib/kubelet /var/lib/etcd /opt/cni/bin; do
  [ -d "$dir" ] && echo "WARN: $dir still exists" || echo "OK: $dir removed"
done

# No Kubernetes iptables rules
sudo iptables -L -n | grep -i kube && echo "WARN: kube iptables rules remain" || echo "OK: iptables clean"
```

---

## Quick Copy — Full Reset Script

For convenience, here's the entire reset as a single script. **Review before running.**

```bash
#!/bin/bash
# kubernetes-full-reset.sh
# WARNING: This will destroy all Kubernetes state on this node.

set -e

echo "=== Resetting kubeadm ==="
sudo kubeadm reset -f 2>/dev/null || true

echo "=== Flushing iptables ==="
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X
sudo ipvsadm --clear 2>/dev/null || true

echo "=== Stopping services ==="
sudo systemctl stop kubelet 2>/dev/null || true
sudo systemctl disable kubelet 2>/dev/null || true
sudo systemctl stop containerd 2>/dev/null || true
sudo systemctl disable containerd 2>/dev/null || true
sudo systemctl stop docker 2>/dev/null || true
sudo systemctl disable docker 2>/dev/null || true

echo "=== Removing packages (RHEL/CentOS) ==="
sudo dnf remove -y kubeadm kubectl kubelet kubernetes-cni cri-tools 2>/dev/null || true
sudo dnf remove -y containerd.io docker-ce docker-ce-cli 2>/dev/null || true

echo "=== Removing data directories ==="
sudo rm -rf /etc/kubernetes ~/.kube /var/lib/kubelet /var/lib/etcd
sudo rm -rf /var/run/kubernetes /var/lib/dockershim
sudo rm -rf /var/lib/containerd /var/lib/docker
sudo rm -rf /opt/cni/bin /etc/cni/net.d
sudo rm -rf /var/log/pods /var/log/kube* /var/log/containers

echo "=== Removing repos ==="
sudo rm -f /etc/yum.repos.d/kubernetes.repo
sudo rm -f /etc/yum.repos.d/docker-ce.repo
sudo dnf clean all 2>/dev/null || true

echo "=== Cleaning up kernel config ==="
sudo rm -f /etc/modules-load.d/kubernetes.conf
sudo rm -f /etc/sysctl.d/kubernetes.conf
sudo sysctl --system

echo "=== Done. Reboot recommended. ==="
```

```bash
chmod +x kubernetes-full-reset.sh
sudo ./kubernetes-full-reset.sh
sudo reboot
```
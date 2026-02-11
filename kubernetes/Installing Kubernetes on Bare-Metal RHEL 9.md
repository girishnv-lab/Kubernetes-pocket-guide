# Installing Kubernetes on Bare-Metal RHEL 9

> **Purpose:** Step-by-step guide to install a production-ready Kubernetes cluster on bare-metal RHEL 9 servers using `kubeadm`, Containerd as the container runtime, and Calico as the CNI plugin.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                          │
│                                                                 │
│  ┌─────────────────────┐   ┌────────────┐   ┌────────────┐     │
│  │    Master Node       │   │  Worker 1  │   │  Worker 2  │     │
│  │                     │   │            │   │            │     │
│  │  • API Server :6443 │   │  • kubelet │   │  • kubelet │     │
│  │  • etcd :2379-2380  │   │  • kube-   │   │  • kube-   │     │
│  │  • Scheduler :10251 │   │    proxy   │   │    proxy   │     │
│  │  • Controller:10252 │   │  • Calico  │   │  • Calico  │     │
│  │  • kubelet :10250   │   │            │   │            │     │
│  │  • Calico           │   │            │   │            │     │
│  └─────────────────────┘   └────────────┘   └────────────┘     │
│                                                                 │
│  Container Runtime: Containerd (systemd cgroup driver)          │
│  CNI Plugin: Calico (pod CIDR: 10.244.0.0/16)                  │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- One or more RHEL 9 machines (minimal install)
- Root or sudo access on all nodes
- Network connectivity between all nodes
- At least 2 CPUs and 2 GB RAM per node (master needs more for production)
- Unique hostname, MAC address, and `product_uuid` on each node

> **Run Steps 1–7 on ALL nodes** (master and workers). Steps 8–9 are master-only. Step 10 is worker-only.

---

## Step 1: Install Kernel Headers

```bash
sudo dnf install kernel-devel-$(uname -r) -y
```

---

## Step 2: Load Required Kernel Modules

Load the modules for the current session:

```bash
sudo modprobe br_netfilter
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe overlay
```

Make them persistent across reboots:

```bash
cat > /etc/modules-load.d/kubernetes.conf << EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
overlay
EOF
```

Verify:

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

---

## Step 3: Configure Sysctl Settings

```bash
cat > /etc/sysctl.d/kubernetes.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

Apply:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1
```

---

## Step 4: Disable Swap

Kubernetes requires swap to be disabled.

```bash
sudo swapoff -a
```

Persist across reboots by commenting out the swap entry in `/etc/fstab`:

```bash
sudo sed -e '/swap/s/^/#/g' -i /etc/fstab
```

Verify:

```bash
free -h
# Swap row should show 0B across the board
```

---

## Step 5: Install Containerd

### Install from Docker Repository

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf makecache
sudo dnf -y install containerd.io
```

### Configure Containerd

Generate the default config and enable the systemd cgroup driver:

```bash
sudo sh -c "containerd config default > /etc/containerd/config.toml"
```

Edit `/etc/containerd/config.toml` and set `SystemdCgroup = true`:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

> **Why SystemdCgroup?** RHEL 9 uses systemd as its init system. The kubelet defaults to the systemd cgroup driver, so Containerd must match — otherwise pods will fail to start with cgroup-related errors.

### Enable and Start

```bash
sudo systemctl enable --now containerd.service
```

Verify:

```bash
sudo systemctl status containerd
sudo ctr version
```

---

## Step 6: Configure Firewall Rules

### Master Node

```bash
sudo firewall-cmd --zone=public --permanent --add-port=6443/tcp      # API Server
sudo firewall-cmd --zone=public --permanent --add-port=2379-2380/tcp  # etcd
sudo firewall-cmd --zone=public --permanent --add-port=10250/tcp      # kubelet API
sudo firewall-cmd --zone=public --permanent --add-port=10251/tcp      # kube-scheduler
sudo firewall-cmd --zone=public --permanent --add-port=10252/tcp      # kube-controller-manager
sudo firewall-cmd --zone=public --permanent --add-port=10255/tcp      # kubelet read-only
sudo firewall-cmd --zone=public --permanent --add-port=5473/tcp       # Calico Typha
sudo firewall-cmd --reload
```

### Worker Nodes

```bash
sudo firewall-cmd --zone=public --permanent --add-port=10250/tcp      # kubelet API
sudo firewall-cmd --zone=public --permanent --add-port=10255/tcp      # kubelet read-only
sudo firewall-cmd --zone=public --permanent --add-port=30000-32767/tcp # NodePort services
sudo firewall-cmd --zone=public --permanent --add-port=5473/tcp       # Calico Typha
sudo firewall-cmd --reload
```

> **Alternative:** If this is a lab/test environment, you can disable the firewall entirely with `sudo systemctl disable --now firewalld`. Not recommended for production.

Verify:

```bash
sudo firewall-cmd --list-ports
```

---

## Step 7: Install Kubernetes Components

### Add the Kubernetes Repository

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

### Install

```bash
sudo dnf makecache
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

### Enable kubelet

```bash
sudo systemctl enable --now kubelet.service
```

> **Note:** The kubelet will crash-loop until `kubeadm init` or `kubeadm join` is run. This is expected.

Verify versions:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

## Step 8: Initialize the Control Plane (Master Node Only)

### Pull Required Images

```bash
sudo kubeadm config images pull
```

### Initialize

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

> **Important:** Save the `kubeadm join` command printed at the end of the output. You'll need it for Step 10.

### Configure kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```bash
kubectl get nodes
# Master should appear with status "NotReady" (CNI not installed yet)
```

---

## Step 9: Install Calico CNI Plugin (Master Node Only)

### Install the Tigera Operator

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

### Configure and Apply Custom Resources

Download the custom resources manifest and update the pod CIDR to match the one used in `kubeadm init`:

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

# Replace the default Calico CIDR with our pod network CIDR
sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.244.0.0\/16/g' custom-resources.yaml

kubectl create -f custom-resources.yaml
```

### Verify Calico is Running

```bash
# Wait for all Calico pods to be ready
kubectl get pods -n calico-system -w
```

Once Calico is fully running, the master node should transition to `Ready`:

```bash
kubectl get nodes
# NAME     STATUS   ROLES           AGE   VERSION
# master   Ready    control-plane   5m    v1.33.x
```

---

## Step 10: Join Worker Nodes

### Generate the Join Command (on master)

If you lost the join command from Step 8, regenerate it:

```bash
sudo kubeadm token create --print-join-command
```

### Run on Each Worker Node

Copy the output from above and run it with `sudo` on each worker:

```bash
sudo kubeadm join <master_ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify (on master)

```bash
kubectl get nodes
# NAME      STATUS   ROLES           AGE   VERSION
# master    Ready    control-plane   10m   v1.33.x
# worker1   Ready    <none>          2m    v1.33.x
# worker2   Ready    <none>          1m    v1.33.x
```

Optionally label your worker nodes:

```bash
kubectl label node worker1 node-role.kubernetes.io/worker=worker
kubectl label node worker2 node-role.kubernetes.io/worker=worker
```

---

## Step 11: Verify with a Test Deployment (Optional)

Deploy a simple NGINX application to confirm the cluster is functional.

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
kubectl apply -f nginx-deployment.yaml
```

Verify:

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get service nginx-service
```

Test access from any node:

```bash
curl http://<worker_node_ip>:30080
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `kubeadm init` fails with "swap is enabled" | Swap not fully disabled | Run `swapoff -a` and check `/etc/fstab` |
| kubelet won't start, cgroup driver mismatch | Containerd not using systemd cgroup | Verify `SystemdCgroup = true` in `/etc/containerd/config.toml`, restart containerd |
| Nodes stuck in `NotReady` | Calico not installed or not running | Check `kubectl get pods -n calico-system` |
| `kubeadm join` fails with token expired | Default tokens expire after 24 hours | Regenerate with `kubeadm token create --print-join-command` |
| Pods stuck in `ContainerCreating` | CNI not configured or firewall blocking | Check Calico pods and firewall rules |
| `connection refused` on port 6443 | API server not running or firewall | Check `kubectl get pods -n kube-system` and firewall rules on master |
| `failed to pull image` | No internet access or DNS issue | Check `nslookup registry.k8s.io` and proxy settings |

---

## Quick Reference — Port Summary

| Port | Component | Required On |
|---|---|---|
| 6443 | API Server | Master |
| 2379-2380 | etcd | Master |
| 10250 | kubelet API | All nodes |
| 10251 | kube-scheduler | Master |
| 10252 | kube-controller-manager | Master |
| 10255 | kubelet read-only | All nodes |
| 5473 | Calico Typha | All nodes |
| 30000-32767 | NodePort services | Workers |
# MongoDB Enterprise Kubernetes pocket guide. 

A single reference for Kubernetes commands, MongoDB operator CRDs, troubleshooting guides, and known issues. Built to help new and existing team members triage Kubernetes support cases faster.
### Would you like to add something? Drop a mail to gireesh.n@mongodb.com or Slack. 

---

## Guides

### Setup & Installation

| Guide | Description |
|---|---|
| [Installing Kubernetes on Bare-Metal RHEL 9](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/kubernetes/Installing%20Kubernetes%20on%20Bare-Metal%20RHEL%209.md) | Full cluster setup with kubeadm, Containerd, and Calico on RHEL 9 |
| [Installing MongoDB Operator with Helm](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/k8%20with%20kind%20and%20helm/Installing%20Operator%20with%20helm.md) | Kind cluster, Helm, and MongoDB Enterprise Operator deployment |
| [Kubernetes Reset & Cleanup](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/kubernetes/Resetting%20and%20Clean%20up%20k8%20cluster.md) | Complete teardown of a Kubernetes node — packages, data, and config |

### TLS & Security

| Guide | Description |
|---|---|
| [Enabling TLS for ReplicaSet](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/Deployment%20CRD/MongoDB%20samples/MongoDB%20Sharded%20Cluster%20with%20TLS.md) | Manual TLS certificate setup for a MongoDB ReplicaSet |
| [ReplicaSet with cert-manager](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/Deployment%20CRD/MongoDB%20samples/MongoDB%20ReplicaSet%20with%20cert-manager%20TLS.md) | Automated TLS using cert-manager with X509 authentication |
| [Sharded Cluster with TLS](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/Deployment%20CRD/MongoDB%20samples/MongoDB%20Sharded%20Cluster%20with%20TLS.md) | TLS for sharded clusters — covers shard, config server, and mongos certs |

### Backup & Operations

| Guide | Description |
|---|---|
| [Ops Manager Snapshots with Filesystem Store](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/Deployment%20CRD/opsManager%20samples/OM%20with%20FS%20snapshot%20store.MD) | NFS-backed backup configuration for Ops Manager |

### Quick Reference

| Guide | Description |
|---|---|
| [kubectl Cheatsheet](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/kubernetes/Kubernetes%20%26%20kubectl%20Cheatsheet.md) | Common commands, shell aliases, and MongoDB-specific one-liners |
| [Log Collection Guide](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/Capturing%20Kubernetes%20logs.MD) | What logs to collect for different failure scenarios |

---



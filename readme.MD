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
| [Enabling TLS for ReplicaSet](https://github.com/girishnv-lab/Kubernetes-pocket-guide/blob/main/kubernetes/Installing%20Kubernetes%20on%20Bare-Metal%20RHEL%209.md) | Manual TLS certificate setup for a MongoDB ReplicaSet |
| [ReplicaSet with cert-manager](docs/mongodb-replicaset-certmanager.md) | Automated TLS using cert-manager with X509 authentication |
| [Sharded Cluster with TLS](docs/mongodb-sharded-cluster-tls.md) | TLS for sharded clusters — covers shard, config server, and mongos certs |

### Backup & Operations

| Guide | Description |
|---|---|
| [Ops Manager Snapshots with Filesystem Store](docs/enabling-opsmanager-snapshots-filesystem-store.md) | NFS-backed backup configuration for Ops Manager |

### Quick Reference

| Guide | Description |
|---|---|
| [kubectl Cheatsheet](docs/kubectl-cheatsheet.md) | Common commands, shell aliases, and MongoDB-specific one-liners |
| [Log Collection Guide](docs/kubernetes-log-collection.md) | What logs to collect for different failure scenarios |

---

## Repo Structure

```
├── README.md
└── docs/
    ├── installing-kubernetes-bare-metal-rhel9.md
    ├── installing-mongodb-operator-helm.md
    ├── kubernetes-reset-cleanup.md
    ├── enabling-tls-mongodb-replicaset-kubernetes.md
    ├── mongodb-replicaset-certmanager.md
    ├── mongodb-sharded-cluster-tls.md
    ├── enabling-opsmanager-snapshots-filesystem-store.md
    ├── kubectl-cheatsheet.md
    └── kubernetes-log-collection.md
```

---

## Contributing

1. Create a new `.md` file in the `docs/` directory
2. Add an entry to the appropriate table in this README
3. Keep guides concise — one topic per document, commands should be copy-pasteable
4. Include a troubleshooting table at the bottom of each guide where applicable


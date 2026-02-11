# MongoDB Enterprise Kubernetes pocket guide. 

A single reference for Kubernetes commands, MongoDB operator CRDs, troubleshooting guides, and known issues. Built to help new and existing team members triage Kubernetes support cases faster.

---

## Guides

### Setup & Installation

| Guide | Description |
|---|---|
| [Installing Kubernetes on Bare-Metal RHEL 9](docs/installing-kubernetes-bare-metal-rhel9.md) | Full cluster setup with kubeadm, Containerd, and Calico on RHEL 9 |
| [Installing MongoDB Operator with Helm](docs/installing-mongodb-operator-helm.md) | Kind cluster, Helm, and MongoDB Enterprise Operator deployment |
| [Kubernetes Reset & Cleanup](docs/kubernetes-reset-cleanup.md) | Complete teardown of a Kubernetes node — packages, data, and config |

### TLS & Security

| Guide | Description |
|---|---|
| [Enabling TLS for ReplicaSet](docs/enabling-tls-mongodb-replicaset-kubernetes.md) | Manual TLS certificate setup for a MongoDB ReplicaSet |
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

### Want to add something? Drop a mail to gireesh.n@mongodb.com or slack. 

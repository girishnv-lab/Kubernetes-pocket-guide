# Enabling Ops Manager Snapshots with Filesystem Backup Store

> **Purpose:** Step-by-step guide to configure Ops Manager backup snapshots using a filesystem-based backup store backed by NFS in a Kubernetes environment.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                     │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │  Ops Manager  │    │ Backup Daemon │                  │
│  │    Pod(s)     │    │    Pod(s)     │                  │
│  │              │    │              │                   │
│  │ /snapshot_   │    │ /snapshot_   │                   │
│  │   store      │    │   store      │                   │
│  └──────┬───────┘    └──────┬───────┘                   │
│         │                   │                           │
│         └─────────┬─────────┘                           │
│                   │                                     │
│            ┌──────┴──────┐                              │
│            │     PVC     │                              │
│            │ (RWX / NFS) │                              │
│            └──────┬──────┘                              │
│                   │                                     │
└───────────────────┼─────────────────────────────────────┘
                    │
             ┌──────┴──────┐
             │  NFS Server  │
             │ /snapshotstore│
             └──────────────┘

  ┌──────────────┐
  │  Oplog DB     │  (Standalone MongoDB for oplog storage)
  │  (oplog-db)   │
  └──────────────┘
```

## Prerequisites

- MongoDB Enterprise Kubernetes Operator deployed and running
- Ops Manager deployed via the Operator
- A dedicated VM or server to act as the NFS server (must be network-reachable from cluster nodes)
- `kubectl` access to the target namespace

---

## Step 1: Set Up the NFS Server

This provides the shared filesystem that both the Ops Manager and Backup Daemon pods will mount.

### Install NFS Packages

```bash
sudo yum install -y nfs-utils
sudo systemctl enable --now nfs-server
sudo systemctl start nfs-server
```

### Configure Firewall

```bash
sudo yum install -y firewalld
sudo systemctl enable --now firewalld

sudo firewall-cmd --add-service=nfs --permanent
sudo firewall-cmd --add-service=mountd --permanent
sudo firewall-cmd --add-service=rpc-bind --permanent
sudo firewall-cmd --reload
```

### Create and Export the Snapshot Directory

```bash
sudo mkdir -p /snapshotstore
sudo chmod 777 /snapshotstore
```

Add the export to `/etc/exports`:

```
/snapshotstore *(rw,sync,no_root_squash,no_subtree_check)
```

Apply and verify:

```bash
sudo exportfs -ra
sudo exportfs -v
```

### Test the NFS Mount (Optional)

From any Kubernetes node, verify the NFS share is accessible:

```bash
mkdir -p /mnt/test-nfs
mount -t nfs <nfs_server_ip>:/snapshotstore /mnt/test-nfs
touch /mnt/test-nfs/testfile && rm /mnt/test-nfs/testfile
umount /mnt/test-nfs
```

> Replace `<nfs_server_ip>` with your NFS server's IP address (e.g., `172.31.45.104`).

---

## Step 2: Create Kubernetes Storage Resources

Create the StorageClass, PersistentVolume, and PersistentVolumeClaim to expose the NFS share to the cluster.

Save the following as `snapshot-storage.yaml`:

```yaml
# StorageClass — placeholder for static provisioning
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: snapshot-sc
provisioner: example.com/placeholder    # Not used — PV is manually provisioned
volumeBindingMode: Immediate
---
# PersistentVolume — points to the NFS share
apiVersion: v1
kind: PersistentVolume
metadata:
  name: snapshotstore-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany                     # Required — multiple pods must mount simultaneously
  persistentVolumeReclaimPolicy: Retain
  storageClassName: snapshot-sc
  nfs:
    path: /snapshotstore
    server: 172.31.45.104               # ← Replace with your NFS server IP
---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: snapshotstore-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: snapshot-sc
  resources:
    requests:
      storage: 10Gi
```

Apply:

```bash
kubectl apply -f snapshot-storage.yaml -n <mdb_namespace>
```

Verify the PVC is bound:

```bash
kubectl get pvc snapshotstore-pvc -n <mdb_namespace>
```

Expected output:

```
NAME                 STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS   AGE
snapshotstore-pvc    Bound    snapshotstore-pv   10Gi       RWX            snapshot-sc    10s
```

> **Important:** The access mode must be `ReadWriteMany` (RWX) because both the Ops Manager pod and the Backup Daemon pod need to mount the same volume simultaneously.

---

## Step 3: Deploy the Oplog Database

The Backup Daemon requires a dedicated MongoDB instance to store oplog data. Deploy a standalone instance:

Save as `oplog-db.yaml`:

```yaml
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: oplog-db
spec:
  version: "6.0.5-ent"
  opsManager:
    configMapRef:
      name: my-project
  credentials: organization-secret
  type: Standalone
```

Apply:

```bash
kubectl apply -f oplog-db.yaml -n <mdb_namespace>
```

Wait for the pod to be ready:

```bash
kubectl get mdb oplog-db -n <mdb_namespace> -w
```

---

## Step 4: Configure the Ops Manager YAML

This is the core step. The Ops Manager custom resource must be updated to:

1. Enable backup with the Backup Daemon
2. Mount the NFS-backed PVC into both the Ops Manager and Backup Daemon pods
3. Reference the filesystem store and oplog database

Save as `ops-manager.yaml`:

```yaml
apiVersion: mongodb.com/v1
kind: MongoDBOpsManager
metadata:
  name: om
spec:
  replicas: 1
  version: "7.0.3"
  adminCredentials: ops-manager-admin-secret

  # Mount the snapshot store into the Ops Manager pod
  statefulSet:
    spec:
      template:
        spec:
          volumes:
            - name: snapshot-store
              persistentVolumeClaim:
                claimName: snapshotstore-pvc
          containers:
            - name: mongodb-ops-manager
              volumeMounts:
                - name: snapshot-store
                  mountPath: /snapshot_store

  backup:
    enabled: true
    members: 2

    # Filesystem store — the name here is referenced in the Ops Manager UI
    fileSystemStores:
      - name: filesystem1

    # Oplog store — points to the standalone MongoDB deployed in Step 3
    opLogStores:
      - name: oplog1
        mongodbResourceRef:
          name: oplog-db
        assignmentLabels:
          - "test1"
          - "test2"

    # Mount the same snapshot store into the Backup Daemon pods
    statefulSet:
      spec:
        template:
          spec:
            volumes:
              - name: snapshot-store
                persistentVolumeClaim:
                  claimName: snapshotstore-pvc
            containers:
              - name: mongodb-backup-daemon
                volumeMounts:
                  - name: snapshot-store
                    mountPath: /snapshot_store

  applicationDatabase:
    members: 3
    version: "6.0.5-ent"
```

### Key Configuration Notes

| Field | Value | Why |
|---|---|---|
| `backup.enabled` | `true` | Turns on the Backup Daemon |
| `backup.members` | `2` | Number of Backup Daemon pods |
| `fileSystemStores[].name` | `filesystem1` | Logical name — you'll configure the path in the Ops Manager UI |
| `mountPath` | `/snapshot_store` | Must match the path configured in Ops Manager UI (Step 6) |
| Volume mount on **both** `mongodb-ops-manager` and `mongodb-backup-daemon` | Same PVC | Both pods need access to the shared filesystem |

Apply:

```bash
kubectl apply -f ops-manager.yaml -n <mdb_namespace>
```

Monitor the rollout:

```bash
kubectl get om om -n <mdb_namespace> -w
kubectl get pods -n <mdb_namespace> -l app=om-backup-daemon -w
```

---

## Step 5: Configure the Filesystem Store in Ops Manager UI

Once Ops Manager and the Backup Daemon pods are running:

1. Log in to the Ops Manager UI
2. Navigate to **Admin** → **Backup** → **File System Stores** (or **Backup** → **Snapshot Storage**)
3. Click **Add New File System Store**
4. Configure:
   - **Name:** `filesystem1` (must match the name in the YAML)
   - **Path:** `/snapshot_store` (must match the `mountPath` in the YAML)
   - **Assignment Labels:** Add labels if needed to match your backup configuration
5. Click **Save**

---

## Step 6: Verify the Setup

### Check Backup Daemon Pods Are Running

```bash
kubectl get pods -n <mdb_namespace> -l app=om-backup-daemon
```

### Check Backup Daemon Logs

```bash
for pod in $(kubectl get pods -n <mdb_namespace> -l app=om-backup-daemon -o name); do
  echo "=== $pod ==="
  kubectl logs $pod -n <mdb_namespace> --container=mongodb-backup-daemon --tail=50
done
```

### Verify the NFS Mount Inside the Pods

```bash
# Check Ops Manager pod
kubectl exec -it om-0 -n <mdb_namespace> --container=mongodb-ops-manager -- df -h /snapshot_store

# Check Backup Daemon pod
kubectl exec -it om-backup-daemon-0 -n <mdb_namespace> --container=mongodb-backup-daemon -- df -h /snapshot_store
```

Both should show the NFS mount.

### Trigger a Test Snapshot

1. In Ops Manager UI, navigate to your deployment
2. Go to **Backup** → **Overview**
3. Click **Start** to enable backup for a replica set
4. Wait for the initial snapshot to complete

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| PVC stuck in `Pending` | NFS server unreachable or PV misconfigured | Verify NFS mount from a cluster node (Step 1). Check PV `nfs.server` IP and `nfs.path`. |
| Backup Daemon pods in `CrashLoopBackOff` | Oplog DB not ready or filesystem store path mismatch | Ensure `oplog-db` is in `Running` state. Verify `mountPath` matches the path in Ops Manager UI. |
| Snapshot fails with `Permission denied` | NFS export permissions | Ensure `/etc/exports` has `no_root_squash` and the directory has `chmod 777`. |
| Ops Manager UI shows "No file system stores" | `fileSystemStores` name mismatch or UI config not completed | Confirm `fileSystemStores[].name` in YAML matches the name configured in the UI. |
| `mount.nfs: access denied by server` | Firewall blocking NFS ports | Rerun the firewall commands from Step 1. Verify with `showmount -e <nfs_server_ip>`. |
| Backup Daemon can't reach oplog DB | Network policy or wrong `mongodbResourceRef` | Verify `oplog-db` pod is running and the service is accessible within the namespace. |
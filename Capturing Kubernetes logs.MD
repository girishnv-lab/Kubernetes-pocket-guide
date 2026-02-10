# Kubernetes Log Collection Guide

> **Purpose:** Step-by-step instructions for collecting diagnostic logs from MongoDB Enterprise Kubernetes deployments. Attach all collected logs (compressed) to the support case.

## Variables

Before running the commands below, replace these placeholders with your actual values:

| Placeholder | Description | Example |
|---|---|---|
| `<mdb_namespace>` | MongoDB namespace | `mongodb` |
| `<failing_pod>` | Name of the failing pod | `my-replica-set-0` |
| `<pod_0>`, `<pod_1>`, `<pod_2>` | Replica set pod names | `my-replica-set-0`, `my-replica-set-1`, `my-replica-set-2` |
| `<opsmanager_pod_N>` | Ops Manager pod names | `ops-manager-0`, `ops-manager-1`, `ops-manager-2` |
| `<backup_daemon_pod>` | Backup daemon pod name | `ops-manager-backup-daemon-0` |
| `<case_number>` | Support case number | `01467493` |

---

## 1. Pod Failures

Collect general cluster state and failing pod details:

```bash
# Cluster state
kubectl get pods -n <mdb_namespace> > allpods.txt
kubectl get sts -n <mdb_namespace> > sts.txt
kubectl get mdb -n <mdb_namespace> -o yaml > mdb.yaml
kubectl get events -n <mdb_namespace> > events.txt

# Failing pod details
kubectl describe pod <failing_pod> -n <mdb_namespace> > failingpoddescribe.txt
kubectl logs <failing_pod> -n <mdb_namespace> > failingpodlog.txt
```

---

## 2. Operator Logs

```bash
kubectl logs deployment/mongodb-enterprise-operator -n <mdb_namespace> > operator.log
```

---

## 3. Ops Manager Pod Logs

Copy Ops Manager application logs (last 2 days) from all Ops Manager pods:

```bash
for pod in <opsmanager_pod_0> <opsmanager_pod_1> <opsmanager_pod_2>; do
  kubectl cp <mdb_namespace>/$pod:/mongodb-ops-manager/logs ./logs_<case_number>/$pod \
    --container=mongodb-ops-manager
done
```

---

## 4. Database Pod Logs

Copy automation agent logs and diagnostic data from each replica set member:

```bash
# Automation agent logs
for pod in <pod_0> <pod_1> <pod_2>; do
  kubectl cp <mdb_namespace>/$pod:/var/log/mongodb-mms-automation ./logs_<case_number>/$pod \
    --container=mongodb-enterprise-database
done

# Diagnostic data (FTDC) — collect from at least one pod
kubectl cp <mdb_namespace>/<pod_0>:/data/diagnostic.data ./logs_<case_number>/<pod_0>/diagnostic.data \
  --container=mongodb-enterprise-database
```

---

## 5. Ops Manager Application Logs (via kubectl cp)

If you need the full Ops Manager logs directory:

```bash
kubectl cp <mdb_namespace>/<opsmanager_pod_0>:/mongodb-ops-manager/logs ./logs_<case_number>/<opsmanager_pod_0> \
  --container=mongodb-ops-manager
```

---

## 6. Backup Daemon Logs

```bash
for pod in <backup_daemon_pod>; do
  kubectl cp <mdb_namespace>/$pod:/mongodb-ops-manager/logs ./logs_<case_number>/$pod \
    --container=mongodb-backup-daemon
done
```

---

## 7. Package and Upload

Compress the collected logs and attach to the support case:

```bash
tar czf logs_<case_number>.tar.gz logs_<case_number>/
```

Upload `logs_<case_number>.tar.gz` to the support case.
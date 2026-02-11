# MongoDB Sharded Cluster with TLS

> **Purpose:** Deploy a TLS-secured MongoDB Sharded Cluster using the MongoDB Enterprise Kubernetes Operator. Covers certificate generation, secret/configmap creation, and the ShardedCluster custom resource.

## Architecture

```
                        ┌─────────────┐
                        │   mongos    │
                        │  (router)   │
                        └──────┬──────┘
                               │
              ┌────────────────┼────────────────┐
              │                                 │
       ┌──────┴──────┐                   ┌──────┴──────┐
       │  Shard 0     │                   │ Config Svr  │
       │  (data)      │                   │ (metadata)  │
       └─────────────┘                   └─────────────┘
```

Each component (shards, config servers, mongos) gets its own TLS secret, all signed by the same CA.

## Prerequisites

- MongoDB Enterprise Kubernetes Operator deployed
- Ops Manager configured and accessible
- TLS certificate generator ([tlsgencer](https://github.com/bhartiroshan/tlsgencer.git))

---

## Hostname Format

Each component type has a distinct hostname pattern:

| Component | Hostname Pattern |
|---|---|
| Shard | `<cluster>-<shard_idx>-<ordinal>.<cluster>-sh.<namespace>.svc.cluster.local` |
| Config Server | `<cluster>-config-<ordinal>.<cluster>-cs.<namespace>.svc.cluster.local` |
| Mongos | `<cluster>-mongos-<ordinal>.<cluster>-svc.<namespace>.svc.cluster.local` |

For a cluster named `my-sharded-cluster` with 1 shard, 1 config server, and 1 mongos:

```
my-sharded-cluster-0-0.my-sharded-cluster-sh.mongodb.svc.cluster.local
my-sharded-cluster-config-0.my-sharded-cluster-cs.mongodb.svc.cluster.local
my-sharded-cluster-mongos-0.my-sharded-cluster-svc.mongodb.svc.cluster.local
```

> **Scaling up?** If you add more shards or members, you must regenerate the certificate with the additional SANs.

---

## Step 1: Generate Certificates

Generate a single certificate with SANs for all components:

```bash
./tlscerlinux -server \
  -host="my-sharded-cluster-0-0.my-sharded-cluster-sh.mongodb.svc.cluster.local,my-sharded-cluster-config-0.my-sharded-cluster-cs.mongodb.svc.cluster.local,my-sharded-cluster-mongos-0.my-sharded-cluster-svc.mongodb.svc.cluster.local" \
  -cn="sharded-cluster"
```

This generates `tlsgencer-server.crt`, `tlsgencer-server.key`, and CA files.

Build the CA PEM (if not already done):

```bash
cat tlsgencer-ca.crt tlsgencer-ia.crt > ca.pem
```

Verify SANs:

```bash
openssl x509 -text -noout -in tlsgencer-server.crt | grep DNS
```

---

## Step 2: Create the CA ConfigMap

```bash
kubectl -n mongodb create configmap sharded-cluster-ca \
  --from-file=ca-pem=ca.pem
```

---

## Step 3: Create TLS Secrets

Each component type requires its own secret. The naming convention is `<certsSecretPrefix>-<component>-cert`.

```bash
# Shard 0
kubectl -n mongodb create secret tls shard-clustertls-my-sharded-cluster-0-cert \
  --cert=tlsgencer-server.crt \
  --key=tlsgencer-server.key

# Config Server
kubectl -n mongodb create secret tls shard-clustertls-my-sharded-cluster-config-cert \
  --cert=tlsgencer-server.crt \
  --key=tlsgencer-server.key

# Mongos
kubectl -n mongodb create secret tls shard-clustertls-my-sharded-cluster-mongos-cert \
  --cert=tlsgencer-server.crt \
  --key=tlsgencer-server.key
```

### Secret Naming Convention

| Component | Secret Name |
|---|---|
| Shard 0 | `<prefix>-<cluster>-0-cert` |
| Shard 1 | `<prefix>-<cluster>-1-cert` |
| Config Server | `<prefix>-<cluster>-config-cert` |
| Mongos | `<prefix>-<cluster>-mongos-cert` |

> **Note:** All three secrets use the same certificate here because the cert contains SANs for all components. In production, you may want separate certificates per component.

Verify:

```bash
kubectl get secrets -n mongodb | grep shard-clustertls
```

---

## Step 4: Deploy the Sharded Cluster

```yaml
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: my-sharded-cluster
  namespace: mongodb
spec:
  type: ShardedCluster
  shardCount: 1
  mongodsPerShardCount: 1
  mongosCount: 1
  configServerCount: 1
  version: "8.0.5-ent"

  opsManager:
    configMapRef:
      name: my-project
  credentials: organization-secret
  persistent: false

  security:
    certsSecretPrefix: shard-clustertls
    authentication:
      enabled: true
      modes: ["X509", "SCRAM"]       # Both auth modes enabled
      agents:
        mode: SCRAM                   # Agent uses SCRAM to communicate with Ops Manager
    tls:
      enabled: true
      ca: sharded-cluster-ca          # ConfigMap from Step 2 (must contain key "ca-pem")

  # --- Resource limits per component ---

  configSrvPodSpec:
    podTemplate:
      spec:
        containers:
          - name: mongodb-enterprise-database
            resources:
              limits:
                cpu: "0.8"
                memory: 1G

  mongosPodSpec:
    podTemplate:
      spec:
        containers:
          - name: mongodb-enterprise-database
            resources:
              limits:
                cpu: "0.8"
                memory: 1G

  shardPodSpec:
    podTemplate:
      spec:
        containers:
          - name: mongodb-enterprise-database
            resources:
              limits:
                cpu: "0.6"
                memory: 3G
    persistent: false
```

Apply:

```bash
kubectl apply -f sharded-cluster.yaml -n mongodb
```

Monitor:

```bash
kubectl get mdb my-sharded-cluster -n mongodb -w
kubectl get pods -n mongodb -w
```

---

## Step 5: Verify

```bash
# Check all pods are running
kubectl get pods -n mongodb -l app=my-sharded-cluster-svc

# Verify TLS on the mongos pod
kubectl exec -it my-sharded-cluster-mongos-0 -n mongodb -- \
  cat /data/automation-mongod.conf | grep -A5 tls

# Check certificate expiry on shard
kubectl get secret shard-clustertls-my-sharded-cluster-0-cert -n mongodb \
  -o jsonpath="{.data.tls\.crt}" | base64 -d | openssl x509 -noout -dates
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Shard pods fail, config/mongos pods fine | Shard secret name mismatch | Must be `<prefix>-<cluster>-<shard_index>-cert` (e.g., `-0-cert` for shard 0) |
| `SSL peer certificate validation failed` | SANs don't cover all components | Regenerate cert with all hostnames in `-host` flag |
| Mongos can't connect to config servers | Config server secret missing or wrong name | Must be `<prefix>-<cluster>-config-cert` |
| Auth fails after enabling X509 | Agent mode not set to SCRAM | Set `agents.mode: SCRAM` so the operator agent can still authenticate |
| Pods stuck in `Pending` | Resource limits too high for nodes | Reduce CPU/memory limits or add nodes |
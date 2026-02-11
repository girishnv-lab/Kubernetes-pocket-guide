# Enabling TLS for MongoDB ReplicaSet in Kubernetes

> **Purpose:** Step-by-step guide to configure TLS encryption for a MongoDB Enterprise ReplicaSet deployed via the MongoDB Kubernetes Operator.

## Prerequisites

- MongoDB Enterprise Kubernetes Operator deployed and running
- Ops Manager configured and accessible
- `kubectl` access to the target namespace
- TLS certificates (server cert, key, CA chain) — either from an internal CA or a public certificate authority

## Variables

| Placeholder | Description | Example |
|---|---|---|
| `<replicaset_name>` | MongoDB ReplicaSet resource name | `k8-rs1` |
| `<mdb_namespace>` | Kubernetes namespace | `mongodb` |
| `<cert_prefix>` | TLS secret prefix used in the MongoDB CR | `myrs-tls` |
| `<members>` | Number of replica set members | `3` |

---

## Step 1: Generate Certificates with Correct SANs

Each pod in a StatefulSet gets a predictable DNS name. Your TLS certificate **must** include a SAN (Subject Alternative Name) entry for every replica set member.

### Hostname Format

```
<replicaset_name>-<ordinal>.<replicaset_name>-svc.<mdb_namespace>.svc.cluster.local
```

### Example — 3-Member ReplicaSet Named `k8-rs1`

The certificate must contain these three SANs:

```
k8-rs1-0.k8-rs1-svc.mongodb.svc.cluster.local
k8-rs1-1.k8-rs1-svc.mongodb.svc.cluster.local
k8-rs1-2.k8-rs1-svc.mongodb.svc.cluster.local
```

### Verify SANs After Generation

```bash
openssl x509 -text -noout -in server.crt | grep DNS
```

Expected output:

```
DNS:k8-rs1-0.k8-rs1-svc.mongodb.svc.cluster.local, DNS:k8-rs1-1.k8-rs1-svc.mongodb.svc.cluster.local, DNS:k8-rs1-2.k8-rs1-svc.mongodb.svc.cluster.local
```

> **Common mistake:** Using ordinal `3` instead of `2` for the third member. A 3-member set uses ordinals `0`, `1`, `2`.

---

## Step 2: Build the CA Certificate Chain

Combine your root CA and any intermediate certificates into a single PEM file. Order matters — **leaf intermediates first, root CA last**.

```bash
cat intermediate-ca.crt root-ca.crt > ca.pem
```

If you only have a single root CA with no intermediates:

```bash
cp root-ca.crt ca.pem
```

---

## Step 3: Create the Kubernetes Secret and ConfigMap

### 3a. Create the TLS Secret

Create a Kubernetes TLS secret containing the server certificate and private key. The secret name **must** follow the naming convention: `<cert_prefix>-<replicaset_name>-cert`.

```bash
kubectl -n <mdb_namespace> create secret tls <cert_prefix>-<replicaset_name>-cert \
  --cert=server.crt \
  --key=server.key
```

**Example:**

```bash
kubectl -n mongodb create secret tls myrs-tls-k8-rs1-cert \
  --cert=server.crt \
  --key=server.key
```

> **Important:** The naming convention `<cert_prefix>-<replicaset_name>-cert` is enforced by the operator. If the secret name doesn't match, the operator will not mount the certificate and TLS will fail silently.

### 3b. Create the CA ConfigMap

Create a ConfigMap containing the CA chain. The key **must** be `ca-pem`.

```bash
kubectl -n <mdb_namespace> create configmap <configmap_name> \
  --from-file=ca-pem=ca.pem
```

**Example:**

```bash
kubectl -n mongodb create configmap rs1-ca \
  --from-file=ca-pem=ca.pem
```

### Verify

```bash
kubectl -n mongodb get secret myrs-tls-k8-rs1-cert
kubectl -n mongodb get configmap rs1-ca
```

---

## Step 4: Update the MongoDB Resource YAML

Reference the secret prefix and CA ConfigMap in your MongoDB custom resource:

```yaml
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: k8-rs1
  namespace: mongodb
spec:
  type: ReplicaSet
  members: 3
  version: "6.0.5-ent"
  opsManager:
    configMapRef:
      name: my-project
  credentials: organization-secret
  persistent: false
  security:
    certsSecretPrefix: myrs-tls          # Must match the prefix used in Step 3a
    tls:
      enabled: true
      ca: rs1-ca                         # ConfigMap name from Step 3b (must contain key "ca-pem")
  # Optional — requireTLS is the default when TLS is enabled
  additionalMongodConfig:
    net:
      ssl:
        mode: "requireTLS"
```

### Key Fields

| Field | Value | Notes |
|---|---|---|
| `security.certsSecretPrefix` | `myrs-tls` | Operator looks for secret named `<prefix>-<replicaset_name>-cert` |
| `security.tls.enabled` | `true` | Enables TLS on the deployment |
| `security.tls.ca` | `rs1-ca` | ConfigMap name; must contain the key `ca-pem` |
| `additionalMongodConfig.net.ssl.mode` | `requireTLS` | Optional — this is already the default |

---

## Step 5: Apply the YAML

```bash
kubectl apply -f mdb.yaml -n mongodb
```

Monitor rollout:

```bash
kubectl -n mongodb get mdb k8-rs1 -w
kubectl -n mongodb get pods -l app=k8-rs1-svc -w
```

---

## Step 6: Verify TLS Configuration Inside the Pod

Once the pods are running, confirm the operator has written the correct TLS config:

```bash
kubectl -n mongodb exec -it k8-rs1-0 -- cat /data/automation-mongod.conf
```

The `net.tls` section should look similar to:

```yaml
net:
  bindIp: 0.0.0.0
  port: 27017
  tls:
    CAFile: /mongodb-automation/tls/ca/ca-pem
    allowConnectionsWithoutCertificates: true
    certificateKeyFile: /mongodb-automation/tls/<hash>
    mode: requireTLS
```

### What to Check

- `tls.mode` is `requireTLS`
- `tls.CAFile` points to `/mongodb-automation/tls/ca/ca-pem`
- `tls.certificateKeyFile` exists (the hash is auto-generated by the operator)

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Pods stuck in `Pending` or `CrashLoopBackOff` after enabling TLS | Secret name doesn't match `<prefix>-<replicaset_name>-cert` | Verify secret naming convention |
| `SSL peer certificate validation failed` | SANs in the certificate don't match pod hostnames | Regenerate cert with correct SANs (Step 1) |
| `no suitable certificate found` | CA chain is incomplete or in wrong order | Rebuild `ca.pem` with full chain (Step 2) |
| `tls` section missing from `automation-mongod.conf` | ConfigMap key is not `ca-pem` | Recreate ConfigMap with `--from-file=ca-pem=ca.pem` |
| Operator logs show `certificate verify failed` | CA ConfigMap doesn't include all intermediates | Add all intermediate certs to `ca.pem` |

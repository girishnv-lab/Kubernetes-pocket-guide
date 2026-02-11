# MongoDB ReplicaSet with cert-manager TLS

> **Purpose:** Deploy a TLS-secured MongoDB ReplicaSet using cert-manager for automatic certificate issuance and renewal.
>
> **Reference:** [MongoDB Kubernetes Operator — cert-manager Integration](https://www.mongodb.com/docs/kubernetes-operator/v1.33/tutorial/cert-manager-integration/)

---

## Prerequisites

- MongoDB Enterprise Kubernetes Operator deployed
- Ops Manager configured and accessible
- `kubectl` access to the target namespace

---

## Step 1: Generate CA Certificates

Clone the TLS certificate generator and create a CA:

```bash
git clone https://github.com/bhartiroshan/tlsgencer.git
cd tlsgencer
./tlscerlinux -server -host=localhost.com,myaws.com -cn=testcert
```

This generates `tlsgenca.crt` and `tlsgenca.key`.

---

## Step 2: Install cert-manager

Follow the internal guide: [Install cert-manager (KB-000021879)](https://knowledge.corp.mongodb.com/article/000021879)

Or install directly:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Verify all cert-manager pods are running:

```bash
kubectl get pods -n cert-manager
```

---

## Step 3: Create the CA Secret

Base64-encode the CA certificate and key:

```bash
base64 -w 0 tlsgenca.crt > ca.crt.b64
base64 -w 0 tlsgenca.key > ca.key.b64
```

Create the secret (replace the `data` values with contents of the `.b64` files):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ca-key-pair
  namespace: mongodb
data:
  tls.crt: <contents-of-ca.crt.b64>
  tls.key: <contents-of-ca.key.b64>
```

```bash
kubectl apply -f ca-secret.yaml
```

---

## Step 4: Build the Ops Manager CA ConfigMap

If Ops Manager uses a custom CA, the Backup Daemon needs additional certificates to download MongoDB binaries from the internet.

### 4a. Download the downloads.mongodb.com Certificate Chain

```bash
openssl s_client -showcerts -verify 2 \
  -connect downloads.mongodb.com:443 -servername downloads.mongodb.com < /dev/null \
  | awk '/BEGIN/,/END/{ if(/BEGIN/){a++}; out="cert"a".crt"; print >out}'
```

This produces `cert1.crt`, `cert2.crt`, `cert3.crt`, etc.

### 4b. Concatenate Into mms-ca.crt

```bash
# Combine your CA cert and key into a PEM
cat tlsgenca.crt tlsgenca.key > tlsgenca.pem

# Append the intermediate and root certs from downloads.mongodb.com
# Skip cert1.crt — that's the server cert, not a CA cert
cat tlsgenca.pem cert2.crt cert3.crt cert4.crt > mms-ca.crt
```

> **Important:** The Kubernetes Operator requires the file to be named `mms-ca.crt` in the ConfigMap.

### 4c. Create the ConfigMap

```bash
kubectl create configmap om-http-cert-ca --from-file="mms-ca.crt" -n mongodb
```

---

## Step 5: Configure cert-manager CA Issuer

Create an Issuer that references the CA secret from Step 3:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: mongodb
spec:
  ca:
    secretName: ca-key-pair
```

```bash
kubectl apply -f ca-issuer.yaml
```

Verify:

```bash
kubectl get issuer ca-issuer -n mongodb
# STATUS should be "True"
```

---

## Step 6: Create the CA ConfigMap for MongoDB

This ConfigMap is referenced by the MongoDB CR. It must contain two keys: `ca-pem` and `mms-ca.crt`.

```bash
kubectl create configmap ca-issuer \
  --from-file=ca-pem=tlsgenca.pem \
  --from-file=mms-ca.crt=mms-ca.crt \
  -n mongodb
```

---

## Step 7: Create Certificates for MongoDB Resources

### 7a. ReplicaSet Server Certificate

The SAN list must include every pod hostname. For a 3-member replica set named `my-replica-set`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-replica-set-certificate
  namespace: mongodb
spec:
  secretName: mdb-my-replica-set-cert
  issuerRef:
    name: ca-issuer
  duration: 240h0m0s
  renewBefore: 120h0m0s
  usages:
    - server auth
    - client auth
  dnsNames:
    - my-replica-set-0
    - my-replica-set-0.my-replica-set-svc.mongodb.svc.cluster.local
    - my-replica-set-1
    - my-replica-set-1.my-replica-set-svc.mongodb.svc.cluster.local
    - my-replica-set-2
    - my-replica-set-2.my-replica-set-svc.mongodb.svc.cluster.local
```

> **Secret naming:** Must follow `<certsSecretPrefix>-<replicaset_name>-cert` — here `mdb-my-replica-set-cert`.

### 7b. MongoDB Agent Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: agent-certs
  namespace: mongodb
spec:
  secretName: mdb-my-replica-set-agent-certs
  issuerRef:
    name: ca-issuer
  commonName: automation
  duration: 12240h0m0s
  renewBefore: 12120h0m0s
  usages:
    - digital signature
    - key encipherment
    - client auth
  dnsNames:
    - automation
  subject:
    countries:
      - US
    provinces:
      - NY
    localities:
      - NY
    organizations:
      - cluster.local-agent
    organizationalUnits:
      - a-1635241837-m5yb81lfnrz
```

> **Note:** The `organizationalUnits` value must match your Ops Manager organization's agent API key ID. Update this to match your environment.

Apply both:

```bash
kubectl apply -f replica-set-cert.yaml
kubectl apply -f agent-cert.yaml
```

Verify certificates are issued:

```bash
kubectl get certificates -n mongodb
# Both should show READY = True
```

---

## Step 8: Deploy the MongoDB ReplicaSet

```yaml
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: my-replica-set
  namespace: mongodb
spec:
  type: ReplicaSet
  members: 3
  version: "8.0.0"
  opsManager:
    configMapRef:
      name: my-project
  credentials: organization-secret
  security:
    certsSecretPrefix: mdb
    authentication:
      enabled: true
      modes:
        - X509
    tls:
      ca: ca-issuer          # ConfigMap from Step 6 (contains ca-pem and mms-ca.crt)
      enabled: true
```

Apply:

```bash
kubectl apply -f mdb-replicaset.yaml -n mongodb
```

Monitor:

```bash
kubectl get mdb my-replica-set -n mongodb -w
kubectl get pods -n mongodb -l app=my-replica-set-svc -w
```

---

## Verification

```bash
# Check certificate expiry
kubectl get secret mdb-my-replica-set-cert -n mongodb \
  -o jsonpath="{.data.tls\.crt}" | base64 -d | openssl x509 -noout -dates

# Check agent certificate expiry
kubectl get secret mdb-my-replica-set-agent-certs -n mongodb \
  -o jsonpath="{.data.tls\.crt}" | base64 -d | openssl x509 -noout -dates

# Verify TLS config inside the pod
kubectl exec -it my-replica-set-0 -n mongodb -- cat /data/automation-mongod.conf | grep -A5 tls
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Certificate stuck in `Issuing` | Issuer not ready or CA secret missing | `kubectl get issuer ca-issuer -n mongodb` — check status and events |
| `mdb-my-replica-set-cert` secret not created | Certificate resource has errors | `kubectl describe certificate my-replica-set-certificate -n mongodb` |
| Agent auth fails with X509 | OU in agent cert doesn't match Ops Manager org | Update `organizationalUnits` to match your Ops Manager agent API key |
| Pods crash with `certificate verify failed` | CA ConfigMap missing `mms-ca.crt` or incomplete chain | Verify `ca-issuer` ConfigMap has both `ca-pem` and `mms-ca.crt` keys |
| Backup Daemon can't download binaries | `om-http-cert-ca` missing internet CA chain | Rebuild `mms-ca.crt` with full chain from downloads.mongodb.com (Step 4) |
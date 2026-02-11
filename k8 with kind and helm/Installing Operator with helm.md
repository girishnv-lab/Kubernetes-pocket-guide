# Installing MongoDB Enterprise Operator with Helm

> **Purpose:** Set up a Kind cluster with Docker, install Helm, and deploy the MongoDB Enterprise Kubernetes Operator.

---

## Step 1: Install Docker

### RHEL / CentOS

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
sudo systemctl enable docker
```

### CentOS (yum)

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

---

## Step 2: Install Kind

```bash
sudo rm -f /usr/bin/kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/bin/kind
kind version
```

---

## Step 3: Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/bin/kubectl
kubectl version --client
```

---

## Step 4: Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

---

## Step 5: Create the Kind Cluster

```bash
curl -Lo kind-config.yaml https://raw.githubusercontent.com/gireesh-nv/mongo_withargo/refs/heads/main/mongoargo_kind.yaml
kind create cluster --config kind-config.yaml --retain --image "kindest/node:v1.35.0"
```

Verify:

```bash
kubectl get nodes
```

---

## Step 6: Deploy the MongoDB Enterprise Operator

```bash
# Add the MongoDB Helm repo
helm repo add mongodb https://mongodb.github.io/helm-charts

# Install the operator into the mongodb namespace
helm install enterprise-operator mongodb/enterprise-operator --namespace mongodb --create-namespace    // Install Kubernetes Operator
helm install mongodb-controller mongodb/mongodb-kubernetes --namespace mongodb --create-namespace --version <MCKO version>  // Install MCKO

# Verify the operator deployment
kubectl describe deployments mongodb-enterprise-operator -n mongodb
```

---

## Step 7: Create the Ops Manager Admin Secret

```bash
kubectl create secret generic ops-manager-admin-secret -n mongodb \
  --from-literal=Username="admin@test.com" \
  --from-literal=Password="Admin@123" \
  --from-literal=FirstName="admin" \
  --from-literal=LastName="admin"
```

> **Note:** Change these credentials for non-lab environments.

---

## Operator Version Management (Helm)

```bash
# List configured Helm repos
helm repo list

# List all available operator versions
helm search repo mongodb/enterprise-operator --versions

# Check currently installed version
helm list -n mongodb

# Upgrade to a specific version
helm upgrade enterprise-operator mongodb/enterprise-operator --namespace mongodb --version 1.24.0

# Install a specific older version (fresh install)
helm install enterprise-operator mongodb/enterprise-operator --namespace mongodb --version 1.18.0

# Rollback to previous release
helm rollback enterprise-operator -n mongodb
```

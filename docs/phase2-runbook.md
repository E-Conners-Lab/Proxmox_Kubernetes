# Phase 2 — Storage, Secrets, and First Workload

## Overview

This runbook picks up from a completed Phase 1 cluster (k3s + Cilium + MetalLB + NGINX Ingress). By the end of Phase 2 you will have:

| Step | Component | What it does |
|------|-----------|--------------|
| 1 | Longhorn | Distributed block storage across worker nodes |
| 2 | HashiCorp Vault | Secrets management with K8s auth + agent injection |
| 3 | ChromaDB | Vector database as the first production workload |
| 4 | Cilium network policies | Least-privilege egress for the agent namespace |
| 5 | Validation | End-to-end test of storage, secrets, and networking |

## Prerequisites

Phase 1 must be complete: 3-node k3s cluster with Cilium, MetalLB, and NGINX Ingress all healthy.

```bash
kubectl get nodes -o wide
# All 3 nodes should show Ready

kubectl get pods -A
# All pods Running, no restarts
```

---

## Step 1 — Install Longhorn

Longhorn provides replicated block storage across worker nodes. Each Longhorn volume is replicated to multiple nodes, so data survives a node failure. PVCs bind to these replicated volumes.

### Verify prerequisites on each worker

`open-iscsi` and `nfs-common` should already be installed from Phase 1. Confirm on each worker:

```bash
ssh <YOUR_USERNAME>@<WORKER1_IP>
systemctl status iscsid.socket
# Should show active (listening)
# iscsid itself will show inactive — that is normal.
# It is socket-activated and starts on demand when
# Longhorn needs an iSCSI connection.

# If not installed:
sudo apt install -y open-iscsi nfs-common
sudo systemctl enable iscsid.socket --now
```

### Install Longhorn via Helm

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultReplicaCount=2 \
  --version 1.10.2
```

> **Note:** `defaultReplicaCount=2` means each volume is replicated across 2 of 3 nodes. This balances redundancy with disk usage for a homelab. Production environments typically use 3.

### Wait for Longhorn to be ready

```bash
kubectl -n longhorn-system get pods -w
# Wait until all pods show Running (can take 5+ minutes)
```

The `-w` flag streams pod status in real time. `ErrImagePull`, `ImagePullBackOff`, and `CrashLoopBackOff` errors during startup are normal. CSI pods race for leader election and restart a few times before settling. Press `Ctrl+C` and check the final state:

```bash
kubectl -n longhorn-system get pods
# All pods should show Running (a few restarts is fine)

kubectl get storageclass
# You should see 'longhorn' listed
```

### Set Longhorn as the default StorageClass

k3s ships with a `local-path` StorageClass that stores data on a single node with no replication. Swap the default to Longhorn so any new PVC automatically gets replicated storage.

```bash
# Remove default from local-path (k3s built-in)
kubectl patch storageclass local-path -p \
  '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Set Longhorn as default
kubectl patch storageclass longhorn -p \
  '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Validate with a test PVC

Create a PVC requesting 1GB of storage, spin up a pod that writes a file to it, and confirm the data is there.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-test-pod
spec:
  containers:
    - name: test
      image: busybox
      command:
        - sh
        - -c
        - echo 'Longhorn works' > /data/test.txt && sleep 3600
      volumeMounts:
        - mountPath: /data
          name: test-vol
  volumes:
    - name: test-vol
      persistentVolumeClaim:
        claimName: longhorn-test
EOF
```

```bash
kubectl wait --for=condition=ready pod/longhorn-test-pod \
  --timeout=120s
kubectl exec longhorn-test-pod -- cat /data/test.txt
# Should print: Longhorn works

# Clean up
kubectl delete pod longhorn-test-pod
kubectl delete pvc longhorn-test
```

---

## Step 2 — Install HashiCorp Vault

Vault provides enterprise-grade secrets management with dynamic secrets, automatic rotation, audit logging, and fine-grained access policies. Pods authenticate using their Kubernetes service accounts and receive secrets via the Vault Agent Injector sidecar.

### Install Vault via Helm

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.dataStorage.storageClass=longhorn \
  --set server.dataStorage.size=5Gi \
  --set injector.enabled=true \
  --set server.dev.enabled=false
```

> **Note:** Running in production mode (`dev.enabled=false`) requires manual initialization and unsealing. This matches how Vault operates in enterprise environments.

### Wait for Vault pod to start

```bash
kubectl -n vault get pods -w
# vault-0 will show 0/1 Running — this is expected
# It needs to be initialized and unsealed before it becomes 1/1
```

### Initialize Vault

Initialization creates the encryption keys that protect everything in Vault. The master key is split into 5 shares using Shamir's Secret Sharing, and any 3 are required to unseal.

```bash
kubectl -n vault exec vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3
```

> **Warning:** This outputs 5 unseal keys and 1 root token. Save ALL of them in a secure location immediately. You need any 3 of the 5 unseal keys to unseal Vault after every restart. If you lose the keys, you lose access to all secrets.

### Unseal Vault

Run this command 3 times, using a different unseal key each time:

```bash
kubectl -n vault exec vault-0 -- \
  vault operator unseal <UNSEAL_KEY_1>
kubectl -n vault exec vault-0 -- \
  vault operator unseal <UNSEAL_KEY_2>
kubectl -n vault exec vault-0 -- \
  vault operator unseal <UNSEAL_KEY_3>

# Verify — Sealed should now show false
kubectl -n vault exec vault-0 -- vault status
```

### Enable the KV secrets engine

KV version 2 provides versioned key-value storage with a history of changes for audit and rollback.

```bash
# Log in with the root token
kubectl -n vault exec vault-0 -- \
  vault login <ROOT_TOKEN>

# Enable KV v2 secrets engine
kubectl -n vault exec vault-0 -- \
  vault secrets enable -path=secret kv-v2
```

### Store your first secret

```bash
kubectl -n vault exec vault-0 -- vault kv put \
  secret/agent/config \
  anthropic-api-key=<YOUR_ANTHROPIC_API_KEY>

# Verify it was stored
kubectl -n vault exec vault-0 -- \
  vault kv get secret/agent/config
```

### Configure Kubernetes authentication

This allows pods to authenticate to Vault using their Kubernetes service account tokens. No hardcoded credentials needed.

```bash
# Enable the Kubernetes auth method
kubectl -n vault exec vault-0 -- \
  vault auth enable kubernetes

# Configure it to talk to the K8s API
kubectl -n vault exec vault-0 -- sh -c '
  vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
'
```

> **Note:** This uses the `KUBERNETES_PORT_443_TCP_ADDR` environment variable injected into every pod. An alternative is `https://kubernetes.default.svc:443` which uses in-cluster DNS. Both are valid.

### Create a Vault policy for the agent

```bash
kubectl -n vault exec vault-0 -- \
  vault policy write agent-policy - <<EOF
path "secret/data/agent/*" {
  capabilities = ["read"]
}
EOF
```

> **Note:** This policy grants read-only access to secrets under `secret/agent/`. Pods can read API keys but cannot modify or delete them.

### Create the agent namespace and service account

```bash
# Create the namespace where the agent stack will run
kubectl create namespace agent

# Create a service account for the agent
kubectl -n agent create serviceaccount agent-sa
```

### Bind the Vault role to the service account

```bash
kubectl -n vault exec vault-0 -- \
  vault write auth/kubernetes/role/agent-role \
    bound_service_account_names=agent-sa \
    bound_service_account_namespaces=agent \
    policies=agent-policy \
    ttl=1h
```

> **Note:** This binds the `agent-sa` service account in the `agent` namespace to `agent-policy`. Any pod running as `agent-sa` can authenticate to Vault and read secrets under `secret/agent/`. The TTL controls how long the token is valid before the pod must re-authenticate.

### Test the Vault Agent Injector

Deploy a test pod with Vault annotations. The injector adds a sidecar that authenticates to Vault and writes the secret to a file inside the pod. The application reads a file instead of talking to Vault directly.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: vault-test
  namespace: agent
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "agent-role"
    vault.hashicorp.com/agent-inject-secret-config:
      "secret/data/agent/config"
spec:
  serviceAccountName: agent-sa
  containers:
    - name: test
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
EOF
```

```bash
# Wait for pod (2/2 — the injector adds a sidecar)
kubectl -n agent wait --for=condition=ready \
  pod/vault-test --timeout=120s

# Verify secret was injected
kubectl -n agent exec vault-test -c test -- \
  cat /vault/secrets/config
# Should show the Anthropic API key

# Clean up
kubectl -n agent delete pod vault-test
```

---

## Step 3 — Deploy ChromaDB

ChromaDB is the vector database that stores knowledge embeddings. Deploying it as a Deployment with a Longhorn PVC ensures data persists across pod restarts and node failures.

### Deploy ChromaDB

Three manifests: a PVC requesting 10GB of Longhorn storage, a Deployment running the ChromaDB container, and a ClusterIP Service for internal access on port 8000.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: chromadb-data
  namespace: agent
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
EOF
```

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chromadb
  namespace: agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chromadb
  template:
    metadata:
      labels:
        app: chromadb
    spec:
      containers:
        - name: chromadb
          image: chromadb/chroma:latest
          ports:
            - containerPort: 8000
          volumeMounts:
            - mountPath: /chroma/chroma
              name: data
          resources:
            requests:
              memory: 512Mi
              cpu: 250m
            limits:
              memory: 1Gi
              cpu: 500m
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: chromadb-data
EOF
```

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: chromadb
  namespace: agent
spec:
  selector:
    app: chromadb
  ports:
    - port: 8000
      targetPort: 8000
  type: ClusterIP
EOF
```

### Verify ChromaDB is running

```bash
kubectl -n agent get pods -w
# chromadb pod should show 1/1 Running

# Test the API from inside the cluster
kubectl -n agent run curl-test --rm -it \
  --image=curlimages/curl --restart=Never -- \
  curl -s http://chromadb:8000/api/v1/heartbeat
# Should return a JSON response with a heartbeat timestamp
```

---

## Step 4 — Cilium Network Policies

Lock down the agent namespace so pods can only reach the services they need. This enforces least-privilege egress.

### Apply the network policy

This `CiliumNetworkPolicy` defines an explicit whitelist. Everything not listed is denied by default.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: agent-egress
  namespace: agent
spec:
  endpointSelector:
    matchLabels:
      app: agent
  egress:
    # Allow DNS resolution
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
    # Allow HTTPS to Anthropic API
    - toFQDNs:
        - matchName: api.anthropic.com
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    # Allow SSH to ContainerLab VM
    - toCIDR:
        - <CONTAINERLAB_VM_IP>/32
      toPorts:
        - ports:
            - port: "2201"
              protocol: TCP
            - port: "2202"
              protocol: TCP
            - port: "2211"
              protocol: TCP
            - port: "2212"
              protocol: TCP
            - port: "2213"
              protocol: TCP
            - port: "2214"
              protocol: TCP
    # Allow access to ChromaDB
    - toEndpoints:
        - matchLabels:
            app: chromadb
      toPorts:
        - ports:
            - port: "8000"
              protocol: TCP
    # Allow access to Vault
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: vault
      toPorts:
        - ports:
            - port: "8200"
              protocol: TCP
EOF
```

> **Note:** Replace `<CONTAINERLAB_VM_IP>` with the IP of the VM running ContainerLab. Adjust SSH ports if your port mappings differ. The DNS rule targets pods labeled `k8s-app: kube-dns`, which matches the default CoreDNS deployment in k3s. If your cluster uses different labels for DNS, adjust the `toEndpoints` match accordingly.

### Test the policy

Deploy a test pod with the `agent` label and verify egress rules:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: policy-test
  namespace: agent
  labels:
    app: agent
spec:
  serviceAccountName: agent-sa
  containers:
    - name: test
      image: curlimages/curl
      command: ["sh", "-c", "sleep 3600"]
EOF
```

```bash
kubectl -n agent wait --for=condition=ready \
  pod/policy-test --timeout=60s

# This should SUCCEED (Anthropic API is allowed)
kubectl -n agent exec policy-test -- \
  curl -s -o /dev/null -w "%{http_code}" \
  https://api.anthropic.com/v1/messages

# This should FAIL / timeout (Google is not allowed)
kubectl -n agent exec policy-test -- \
  curl -s --max-time 5 https://www.google.com

# This should SUCCEED (ChromaDB is allowed)
kubectl -n agent exec policy-test -- \
  curl -s http://chromadb:8000/api/v1/heartbeat

# Clean up
kubectl -n agent delete pod policy-test
```

Google timing out while Anthropic and ChromaDB succeed confirms the network policy is working correctly.

---

## Step 5 — End-to-End Validation

### Check all components

```bash
# Cluster nodes
kubectl get nodes -o wide

# All pods across all namespaces
kubectl get pods -A

# Longhorn volumes
kubectl -n longhorn-system get volumes.longhorn.io

# Vault status
kubectl -n vault exec vault-0 -- vault status
# Sealed: false, HA Enabled: false

# ChromaDB health
kubectl -n agent get pods -l app=chromadb

# Network policies
kubectl -n agent get ciliumnetworkpolicies
```

### Summary

| Component | Namespace | Status check |
|-----------|-----------|--------------|
| Longhorn storage | longhorn-system | `kubectl get sc` (longhorn = default) |
| HashiCorp Vault | vault | `vault status` shows Sealed: false |
| ChromaDB | agent | Heartbeat API returns 200 |
| Network policies | agent | Google blocked, Anthropic allowed |
| Agent service account | agent | agent-sa bound to Vault role |

---

## Quick Reference — Placeholder Values

Fill in your values before starting. These carry forward from Phase 1.

| Placeholder | Description |
|-------------|-------------|
| `<YOUR_USERNAME>` | SSH username for cluster VMs |
| `<SERVER_IP>` | k3s server node IP |
| `<WORKER1_IP>` | k3s worker-01 IP |
| `<WORKER2_IP>` | k3s worker-02 IP |
| `<ROOT_TOKEN>` | Vault root token (from init) |
| `<UNSEAL_KEY_1-5>` | Vault unseal keys (save all 5) |
| `<YOUR_ANTHROPIC_API_KEY>` | Anthropic API key |
| `<CONTAINERLAB_VM_IP>` | IP of VM running ContainerLab |

> **Warning:** Vault unseal keys and the root token are the most critical secrets in this build. Store them in a password manager or a physically secure location. If you lose the unseal keys, all secrets stored in Vault become permanently inaccessible.

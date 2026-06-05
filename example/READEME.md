# HashiCorp Vault & External Secrets Setup Guide

This guide describes how to configure HashiCorp Vault, setup the External Secrets Operator, and apply the Kubernetes configurations for both the application workloads and ArgoCD.

---

## 1. Configure HashiCorp Vault

### Enable KV-v2 Secrets Engine
```bash
vault secrets enable -path=quests-tracker kv-v2
```

### Write Secrets to Vault

#### Application Secrets
```bash
vault kv put quests-tracker/quests-tracker-secrets \
  DATABASE_URL="postgresql://postgres:CHqQHho8G2yrmEXawyc87bn3@postgres:5432/quests_tracker_db" \
  JWT_ADVENTURER_SECRET="0PlJOVLc9QzidUoj3BA7ZLX1" \
  JWT_ADVENTURER_REFRESH_SECRET="LmUS4Sgko3GSg07yksrSeQGo" \
  JWT_GUILD_COMMANDER_SECRET="M2t4p3jS7qY0eF1oX8jT5wR0" \
  JWT_GUILD_COMMANDER_REFRESH_SECRET="P5c2d6jX1tA4yR7oW3dZ8qV5"
```

#### Postgres Secrets
```bash
vault kv put quests-tracker/postgres-secrets \
  POSTGRES_USER="postgres" \
  POSTGRES_DB="quests_tracker_db" \
  POSTGRES_PASSWORD="CHqQHho8G2yrmEXawyc87bn3"
```

#### ArgoCD Repo Credentials (GitHub PAT)
```bash
vault kv put quests-tracker/argocd \
  GITHUB_ACCESS_TOKEN="<github-token>" \
  GITHUB_USERNAME="<github-username>"
```

### Configure Policies & Kubernetes Auth

1. **Write Vault Policy**:
   ```bash
   vault policy write read-only-policy - <<EOF
   path "*" {
     capabilities = ["read"]
   }
   EOF
   ```

2. **Configure Kubernetes Auth Role**:
   Ensure you have configured Kubernetes authentication in Vault (configure token reviewer), then bound the backend service account to the policy:
   ```bash
   vault write auth/kubernetes/role/vault-cluster-ss \
     bound_service_account_names=vault-backend-sa \
     bound_service_account_namespaces=vault \
     token_policies=read-only-policy \
     token_ttl=1h
   ```

---

## 2. Deploy infrastructure on Kubernetes

### Extract Vault CA Certificate & Create K8s Secret
```bash
openssl s_client -connect 192.168.1.207:8200 -showcerts </dev/null 2>/dev/null \
  | sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' > vault_ca.crt

kubectl create secret generic vault-ca-secret \
  --from-file=ca.crt=vault_ca.crt \
  --namespace=default
```

### Apply Namespaces & Service Accounts
```bash
kubectl apply -f app-namespace.yml
kubectl apply -f vault-sa.yml
```

### Apply External Secrets Store & Resources
```bash
# ClusterSecretStore (points to quests-tracker path in Vault)
kubectl apply -f secret-store.yml

# Application ExternalSecrets (fetches postgres-secrets and quests-tracker-secrets)
kubectl apply -f external-secret.yml
```

### Verify External Secrets Sync
```bash
# Verify the Store is Ready
kubectl get clustersecretstore

# Verify Secrets are synced into quest-tracker-app namespace
kubectl get externalsecret -n quest-tracker-app
```

---

## 3. Deploy ArgoCD Applications

### Deploy ArgoCD Repository Credentials
```bash
kubectl apply -f argocd/external-secret.yml
```

### Apply ArgoCD Applications
```bash
# Postgres Application
kubectl apply -f argocd/postgres-app.yaml

# API Application
kubectl apply -f argocd/quests-tracker-app.yml
```

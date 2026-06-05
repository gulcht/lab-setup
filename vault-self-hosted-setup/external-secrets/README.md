# HashiCorp Vault & Kubernetes Integration Guide

This repository contains configuration files and a step-by-step guide to integrate **HashiCorp Vault** with a **Kubernetes** cluster using the **External Secrets Operator (ESO)**. This enables Kubernetes workloads to securely retrieve secrets managed by Vault.

---

## File Contents

This repository includes the following configuration files:
*   [vault-token-reviewer.yml](./vault-token-reviewer.yml): Defines the `Namespace`, `ServiceAccount` (`vault-auth`), and `ClusterRoleBinding` for Vault's token review mechanism.
*   [service-account.yml](./service-account.yml): Defines the `Namespace` (`example`) and target `ServiceAccount` (`my-sa`) for running workloads.
*   [secret-store.yml](./secret-store.yml): Defines the External Secrets `SecretStore` pointing to Vault.
*   [external-secret.yml](./external-secret.yml): Defines the `ExternalSecret` resource that fetches secrets from Vault.

---

## Step-by-Step Setup Guide

### Step 1: Set Up Vault Token Reviewer in Kubernetes
Vault needs permissions to verify service account tokens with the Kubernetes API. Apply the token reviewer manifest:

```bash
kubectl apply -f vault-token-reviewer.yml
```

### Step 2: Retrieve Kubernetes Cluster Information
You need to extract the API Server endpoint and the CA Certificate from your active Kubernetes context.

1.  **Get the Kubernetes Cluster API Server URL:**
    ```bash
    kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
    ```

2.  **Get the Kubernetes Cluster CA Certificate:**
    ```bash
    kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > k8s_ca.crt
    ```

3.  **Generate a Service Account Token for Vault Auth:**
    ```bash
    # Note: Default token expires in 1 hour. To extend it (e.g. 14 days / 336 hours):
    kubectl create token vault-auth -n vault --duration=336h
    ```

    > [!WARNING]
    > Tokens generated via `kubectl create token` are temporary (defaulting to 1 hour, or extended with `--duration`). Once they expire, Vault's `token_reviewer_jwt` will fail with a `403 permission denied` error.
    >
    > To create a permanent token (no expiration), you can generate a legacy ServiceAccount token secret:
    > ```yaml
    > apiVersion: v1
    > kind: Secret
    > metadata:
    >   name: vault-auth-token
    >   namespace: vault
    >   annotations:
    >     kubernetes.io/service-account.name: "vault-auth"
    > type: kubernetes.io/service-account-token
    > ```
    > Extract it using: `kubectl get secret vault-auth-token -n vault -o jsonpath='{.data.token}' | base64 -d`


---

### Step 3: Configure Kubernetes Authentication in Vault
Log in to your Vault server and configure the Kubernetes authentication backend.

1.  **Log in to the Vault CLI:**
    ```bash
    vault login
    ```

2.  **Enable the Kubernetes Auth Method:**
    ```bash
    vault auth enable kubernetes
    ```

3.  **Configure Kubernetes Auth Connection:**
    ```bash
    vault write auth/kubernetes/config \
        token_reviewer_jwt="<your-vault-auth-token-from-step-2.3>" \
        kubernetes_host="<your-k8s-cluster-api-server-from-step-2.1>" \
        kubernetes_ca_cert=@k8s_ca.crt
    ```

---

### Step 4: Configure Secrets & Policies in Vault

1.  **Enable KV Secrets Engine (v2):**
    ```bash
    vault secrets enable -path=secret kv-v2
    ```

2.  **Create a Sample Secret:**
    ```bash
    vault kv put secret/myapp username="admin" password="P@ssw0rd123"
    ```

3.  **Create a Read-Only Policy for External Secrets:**
    ```bash
    vault policy write external-secrets-policy - <<EOF
    path "secret/data/myapp" {
      capabilities = ["read"]
    }
    EOF
    ```

---

### Step 5: Configure TLS Certificate & Workload on Kubernetes

1.  **Retrieve Vault's Server Certificate (if using self-signed TLS):**
    ```bash
    openssl s_client -connect 192.168.1.207:8200 -showcerts </dev/null 2>/dev/null \
      | sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' > vault_ca.crt
    ```

2.  **Create Kubernetes Secret for Vault CA:**
    ```bash
    kubectl create secret generic vault-ca-secret -n example \
      --from-file=ca.crt=vault_ca.crt
    ```

3.  **Create Workload Namespace and Service Account:**
    ```bash
    kubectl apply -f service-account.yml
    ```

4.  **Create a Kubernetes Role in Vault for the Workload:**
    Map the target Service Account (`my-sa` in namespace `example`) to the read policy:
    ```bash
    vault write auth/kubernetes/role/demo \
      bound_service_account_names=my-sa \
      bound_service_account_namespaces=example \
      token_policies=external-secrets-policy \
      token_ttl=1h
    ```

---

### Step 6: Deploy External Secrets Resources

1.  **Deploy the SecretStore:**
    ```bash
    kubectl apply -f secret-store.yml
    ```

2.  **Verify the SecretStore Status:**
    ```bash
    kubectl get secretstore -n example
    ```

3.  **Deploy the ExternalSecret:**
    ```bash
    kubectl apply -f external-secret.yml
    ```

4.  **Verify the ExternalSecret Sync Status:**
    ```bash
    kubectl get externalsecret -n example
    ```

---

### Step 7: Verify Synchronized Secrets
Verify that the secret was correctly fetched from Vault and injected as a native Kubernetes Secret.

1.  **Get raw base64-encoded secret data:**
    ```bash
    kubectl get secret myapp-all-secrets -n example -o jsonpath='{.data}' | jq
    ```

2.  **Retrieve decoded secret key-value pairs:**
    ```bash
    kubectl get secret myapp-all-secrets -n example -o json \
      | jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'
    ```

---

## Sources & References

*   [OneUptime Blog: External Secrets Operator with HashiCorp Vault](https://oneuptime.com/blog/post/2026-02-09-external-secrets-operator-hashicorp-vault/view)
*   [Medium: External Secrets Operator with Vault](https://medium.com/@topahadzi/external-secret-operator-with-vault-a781be1048a1)
*   [External Secrets Operator Documentation (HashiCorp Vault Provider)](https://external-secrets.io/v0.5.6/provider-hashicorp-vault/)
*   [HashiCorp Vault Documentation: Kubernetes Auth Method](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
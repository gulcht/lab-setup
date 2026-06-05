# Install Vault

> [!NOTE]
> This setup is for running a self-hosted Vault server **externally (outside the Kubernetes cluster)**.

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```

---

## 🚀 Run as a systemd Service (Persistent & Recommended)

When Vault is installed via the HashiCorp repository, a system user/group named `vault` and a systemd service are automatically created. Storing Vault data in `/tmp` is risky because `/tmp` is wiped on reboot. Follow these steps to set up Vault as a persistent systemd service.

### 1. Create Persistent Directories & Set Permissions:
```bash
# Create directories for data and certificates
sudo mkdir -p /var/lib/vault/data
sudo mkdir -p /etc/vault.d/certs

# Set ownership to the vault user
sudo chown -R vault:vault /var/lib/vault
sudo chown -R vault:vault /etc/vault.d
```

### 2. Generate TLS Certificates:
Generate certificates and place them in the secure `/etc/vault.d/certs/` directory:
```bash
sudo openssl req -x509 -newkey rsa:4096 -sha256 -days 365 \
    -nodes -keyout /etc/vault.d/certs/vault-key.pem -out /etc/vault.d/certs/vault-cert.pem \
    -subj "/CN=localhost" \
    -addext "subjectAltName=DNS:localhost,IP:192.168.1.207"

# Secure the private key
sudo chmod 600 /etc/vault.d/certs/vault-key.pem
sudo chown -R vault:vault /etc/vault.d/certs
```

### 3. Create Server Configuration:
Save the configuration to the standard `/etc/vault.d/vault.hcl` path:
```bash
sudo tee /etc/vault.d/vault.hcl << EOF
api_addr                = "https://192.168.1.207:8200"
cluster_addr            = "https://192.168.1.207:8201"
cluster_name            = "learn-vault-cluster"
disable_mlock           = true
ui                      = true

listener "tcp" {
  address       = "192.168.1.207:8200"
  tls_cert_file = "/etc/vault.d/certs/vault-cert.pem"
  tls_key_file  = "/etc/vault.d/certs/vault-key.pem"
}

backend "raft" {
  path    = "/var/lib/vault/data"
  node_id = "learn-vault-server"
}
EOF

sudo chown vault:vault /etc/vault.d/vault.hcl

# Create vault.env (required by systemd service, will cause startup failure if missing)
sudo touch /etc/vault.d/vault.env
sudo chown vault:vault /etc/vault.d/vault.env
```

### 4. Configure & Start the systemd Service:
By default, the `vault` package comes with `/lib/systemd/system/vault.service` configured to read `/etc/vault.d/vault.hcl`.

Enable and start the service:
```bash
# Reload systemd manager configuration
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable vault

# Start Vault service
sudo systemctl start vault

# Check service status
sudo systemctl status vault
```

---

## 🔑 Initialize and Unseal

### 1. Set environment variables:

**Temporary (for current session):**
```bash
export VAULT_ADDR=https://192.168.1.207:8200
export VAULT_SKIP_VERIFY=true
```

**Permanent (add to `~/.bashrc`):**
```bash
echo 'export VAULT_ADDR=https://192.168.1.207:8200' >> ~/.bashrc
echo 'export VAULT_SKIP_VERIFY=true' >> ~/.bashrc
source ~/.bashrc
```

> [!WARNING]
> Do not skip TLS verification (`VAULT_SKIP_VERIFY=true`) in a production environment.



### 2. Initialize Vault:
```bash
vault operator init -key-shares=1 -key-threshold=1
```
* Note the returned Unseal Key and Root Token (e.g.):
* Unseal Key 1: `<your-unseal-key>`
* Initial Root Token: `<your-root-token>`

### 3. Unseal the server:
```bash
vault operator unseal
```
*Paste the unseal key when prompted.*

---

## 📊 Check Server Status

```bash
$ vault status
```

---

## 🔗 Source
- [HashiCorp Vault Tutorial: Get Started Setup](https://developer.hashicorp.com/vault/tutorials/get-started/setup)
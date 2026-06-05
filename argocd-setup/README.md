# ArgoCD Setup Guide

This guide provides step-by-step instructions for installing, configuring, and accessing ArgoCD on a Kubernetes cluster.

---

## 📋 Prerequisites

Before you begin, ensure you have:
- A running Kubernetes cluster.
- `kubectl` and `helm` CLIs installed and configured.
- Sufficient privileges on the cluster.

### Install Traefik Ingress Controller
If you don't have Traefik installed, set it up using the following commands:

```bash
# Add Traefik Helm repository
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install Traefik
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace
```

Verify the installation:
```bash
# Check if Traefik pod is running
kubectl get pods -n traefik

# Get the Traefik service details (to find the External IP)
kubectl get svc -n traefik
```

---


## 🚀 Installation Steps

### 1. Create the Namespace
ArgoCD resources are isolated in their own namespace. Create the `argocd` namespace:

```bash
kubectl create namespace argocd
```

### 2. Deploy ArgoCD
Apply the official manifests to install ArgoCD. We use the `--server-side` and `--force-conflicts` flags to handle the large Custom Resource Definitions (CRDs) cleanly:

```bash
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## 🔓 Accessing the ArgoCD UI

### 1. Port Forwarding
To access the ArgoCD API server / Web UI from your local machine, run the following port forwarding command:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

You can now access the Web UI at: [https://localhost:8080](https://localhost:8080)

### 2. Retrieve the Initial Admin Password
The default username is `admin`. The initial password is automatically generated and stored in a Kubernetes secret. Retrieve and decode it using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

### 3. Access via Ingress (Alternative)
Apply the Ingress manifest to expose ArgoCD through an Ingress Controller (e.g., Traefik Ingress Controller):


```bash
kubectl apply -f argocd-ingress.yaml
```

Because Traefik requires standard Ingress annotations to be placed on the Service rather than the Ingress resource, you must also annotate the `argocd-server` Service to skip TLS verification for its self-signed certificate:

```bash
# Tell Traefik to speak HTTPS with the service
kubectl annotate service argocd-server -n argocd traefik.ingress.kubernetes.io/service.serversscheme=https --overwrite

# Link the ServersTransport to the service to skip TLS verification
kubectl annotate service argocd-server -n argocd traefik.ingress.kubernetes.io/service.serverstransport=argocd-argocd-transport@kubernetescrd --overwrite
```

*Note: Remember to update the `host` and `tls.hosts` in [argocd-ingress.yaml](file:///Ubuntu/home/asheq04/workspace/argocd/argocd-setup/argocd-ingress.yaml) to match your domain name.*



---

## 🛠️ Next Steps

- **Change Admin Password**: Once logged in, it is highly recommended to change the admin password or delete the initial admin secret.
- **Install ArgoCD CLI**: Download the `argocd` CLI to manage applications from your terminal.
- **Define Applications**: Start deploying your applications using the GitOps pattern!
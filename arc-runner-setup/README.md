# Actions Runner Controller (ARC) Setup Guide

Deploy ARC to run GitHub self-hosted runners on your Kubernetes cluster.

---

## 📋 Prerequisites

1. **Install cert-manager**:
   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
   ```

2. **Generate GitHub PAT (Personal Access Token)**:
   - Go to [Create new Token](https://github.com/settings/tokens/new).
   - Select the `repo` scope.
   - Generate and copy the token.

---

## 🚀 Deploy and Configure ARC

### 1. Install actions-runner-controller via Helm:
```bash
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
```

```bash
helm upgrade --install --namespace actions-runner-system --create-namespace \
  --set=authSecret.create=true \
  --set=authSecret.github_token="REPLACE_YOUR_TOKEN_HERE" \
  --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```

### 2. Deploy the Runner:
```bash
kubectl apply -f arc-runner.yml
```

---

## 🔗 Source
- [Actions Runner Controller Quickstart](https://github.com/actions/actions-runner-controller/blob/master/docs/quickstart.md)
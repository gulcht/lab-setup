# ArgoCD Vault Setup

```bash
# Create secret in Vault
vault kv put quests-tracker/argocd \
   GITHUB_ACCESS_TOKEN="<github-token>" \
   GITHUB_USERNAME="<github-username>"

# Apply External Secrets
kubectl apply -f external-secret.yml

# Apply Applications
kubectl apply -f postgres-app.yaml
kubectl apply -f quests-tracker-app.yaml
```
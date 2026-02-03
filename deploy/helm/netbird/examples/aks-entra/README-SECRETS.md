# Kubernetes Secrets for Netbird AKS Deployment

This directory contains examples for creating secrets for Netbird deployment on AKS.
For production, use Azure Key Vault with Secrets Store CSI Driver.

## Quick Setup

```bash
# Create namespace
kubectl create namespace netbird

# Create Entra authentication secret
kubectl create secret generic netbird-auth-entra \
  --from-literal=client-id='YOUR_CLIENT_ID' \
  --from-literal=client-secret='YOUR_CLIENT_SECRET' \
  -n netbird

# Create relay secret
kubectl create secret generic netbird-relay-secret \
  --from-literal=relay-secret="$(openssl rand -base64 32)" \
  -n netbird
```

## Security Recommendations

1. **Never commit secrets to Git**
2. **Use Azure Key Vault for production**
3. **Rotate secrets regularly**
4. **Use RBAC to limit access**

See the main README-AKS.md for detailed secret management options.

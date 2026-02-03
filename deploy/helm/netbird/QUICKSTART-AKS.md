# Quick Start: Netbird on AKS with Microsoft Entra

This is a quick reference for deploying Netbird on AKS. For detailed instructions, see [README-AKS.md](README-AKS.md).

## Prerequisites Checklist

- [ ] Azure subscription
- [ ] Azure CLI installed
- [ ] kubectl installed
- [ ] Helm 3.x installed
- [ ] Domain name with DNS access

## Quick Deployment (15 minutes)

### 1. Azure App Registration (5 min)

1. Go to Azure Portal → Microsoft Entra ID → App registrations → New registration
2. Name: "Netbird", Single tenant, No redirect URI yet
3. Copy **Application (client) ID** and **Directory (tenant) ID**
4. Create **Client Secret** and copy the value
5. Add API permissions: `openid`, `profile`, `email`, `offline_access`, `User.Read`
6. Grant admin consent

### 2. Create AKS Cluster (5 min)

```bash
az group create --name netbird-rg --location eastus
az aks create --resource-group netbird-rg --name netbird-aks --node-count 3 --generate-ssh-keys
az aks get-credentials --resource-group netbird-rg --name netbird-aks
```

### 3. Install Prerequisites (3 min)

```bash
# NGINX Ingress
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace

# cert-manager
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true

# ClusterIssuer
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

### 4. Configure and Deploy (2 min)

```bash
# Get ingress IP
INGRESS_IP=$(kubectl get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Point your domain to: $INGRESS_IP"

# Create namespace and secrets
kubectl create namespace netbird
kubectl create secret generic netbird-auth-entra \
  --from-literal=client-id='YOUR_CLIENT_ID' \
  --from-literal=client-secret='YOUR_CLIENT_SECRET' \
  -n netbird
kubectl create secret generic netbird-relay-secret \
  --from-literal=relay-secret="$(openssl rand -base64 32)" \
  -n netbird

# Edit values-aks-entra.yaml and replace:
# - YOUR_DOMAIN.com with your domain
# - YOUR_TENANT_ID with your tenant ID
# - YOUR_CLIENT_ID with your client ID

# Deploy Netbird
helm install netbird . --namespace netbird --values values-aks-entra.yaml
```

### 5. Finalize (2 min)

1. **Update DNS**: Point your domain A record to `$INGRESS_IP`
2. **Update Azure App**: Add redirect URI: `https://YOUR_DOMAIN.com/auth/callback`
3. **Access Dashboard**: Navigate to `https://YOUR_DOMAIN.com`

## Configuration Placeholders to Replace

In `values-aks-entra.yaml`:

| Placeholder | Replace With | Where to Find |
|------------|--------------|---------------|
| `YOUR_DOMAIN.com` | Your domain (e.g., `netbird.example.com`) | Your DNS provider |
| `YOUR_TENANT_ID` | Azure Directory (tenant) ID | Azure Portal → App Registration → Overview |
| `YOUR_CLIENT_ID` | Azure Application (client) ID | Azure Portal → App Registration → Overview |

In secrets (via kubectl):
- `YOUR_CLIENT_SECRET`: From Azure Portal → App Registration → Certificates & secrets

## Verification Commands

```bash
# Check pods
kubectl get pods -n netbird

# Check ingress
kubectl get ingress -n netbird

# Check certificate
kubectl get certificate -n netbird

# View logs
kubectl logs -l app.kubernetes.io/component=management -n netbird
```

## Troubleshooting

| Issue | Quick Fix |
|-------|-----------|
| Pods not starting | `kubectl describe pod <pod-name> -n netbird` |
| TLS certificate pending | Wait 2-3 minutes, check `kubectl describe certificate netbird-tls -n netbird` |
| 404 on domain | Verify DNS propagation, check ingress controller |
| Auth errors | Verify secrets are created correctly, check client ID/secret |

## Next Steps

- Configure firewall rules
- Set up monitoring
- Configure backup for persistent volumes
- Review security recommendations in README-AKS.md
- Set up Azure Key Vault for secrets management

## Support

For detailed troubleshooting and configuration options, see [README-AKS.md](README-AKS.md).

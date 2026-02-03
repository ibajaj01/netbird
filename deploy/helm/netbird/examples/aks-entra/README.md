# AKS with Microsoft Entra Example

This directory contains example configurations for deploying Netbird on Azure Kubernetes Service (AKS) with Microsoft Entra (Azure AD) authentication.

## Files

- `../../values-aks-entra.yaml` - Complete values file pre-configured for AKS with Entra authentication
- `README-SECRETS.md` - Guide for creating and managing secrets

## Quick Deploy

1. **Create secrets**:
   ```bash
   kubectl create namespace netbird
   kubectl create secret generic netbird-auth-entra \
     --from-literal=client-id='YOUR_CLIENT_ID' \
     --from-literal=client-secret='YOUR_CLIENT_SECRET' \
     -n netbird
   kubectl create secret generic netbird-relay-secret \
     --from-literal=relay-secret="$(openssl rand -base64 32)" \
     -n netbird
   ```

2. **Update values-aks-entra.yaml**:
   - Replace `YOUR_DOMAIN.com` with your domain
   - Replace `YOUR_TENANT_ID` with your Azure tenant ID
   - Replace `YOUR_CLIENT_ID` with your Azure client ID

3. **Deploy**:
   ```bash
   helm install netbird ../../ \
     --namespace netbird \
     --values ../../values-aks-entra.yaml
   ```

## Full Documentation

See the complete deployment guide: [README-AKS.md](../../README-AKS.md)

## Prerequisites

- AKS cluster running
- NGINX Ingress Controller installed
- cert-manager installed
- Azure App Registration configured
- Domain name configured

## Next Steps

After deployment:
1. Update DNS to point to ingress IP
2. Update Azure App Registration redirect URI
3. Access dashboard at https://YOUR_DOMAIN.com
4. Configure Netbird settings

## Support

For issues or questions, see the troubleshooting section in [README-AKS.md](../../README-AKS.md).

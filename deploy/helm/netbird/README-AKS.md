# Netbird Deployment on Azure Kubernetes Service (AKS) with Microsoft Entra Authentication

This guide provides step-by-step instructions for deploying Netbird on Azure Kubernetes Service (AKS) with Microsoft Entra (formerly Azure Active Directory) authentication.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Azure Setup](#azure-setup)
  - [1. Create AKS Cluster](#1-create-aks-cluster)
  - [2. Configure Microsoft Entra App Registration](#2-configure-microsoft-entra-app-registration)
  - [3. Configure DNS](#3-configure-dns)
- [Deployment Steps](#deployment-steps)
  - [1. Install Prerequisites](#1-install-prerequisites)
  - [2. Configure Values](#2-configure-values)
  - [3. Create Secrets](#3-create-secrets)
  - [4. Deploy Netbird](#4-deploy-netbird)
- [Post-Deployment Configuration](#post-deployment-configuration)
- [Verification](#verification)
- [Security Recommendations](#security-recommendations)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)

## Prerequisites

- Azure subscription with appropriate permissions
- Azure CLI installed (`az` command)
- `kubectl` installed
- `helm` 3.x installed
- A domain name with ability to configure DNS records
- Basic understanding of Kubernetes and Azure services

## Architecture Overview

Netbird consists of four main components:

1. **Management Service**: Central control plane for managing users, peers, and policies
2. **Signal Service**: Facilitates WebRTC signaling for P2P connections
3. **Relay Service**: Fallback relay for NAT traversal (TURN/STUN)
4. **Dashboard**: Web UI for administration

All components are deployed as separate services in Kubernetes, exposed through an Ingress controller with TLS termination.

## Azure Setup

### 1. Create AKS Cluster

Create an AKS cluster with the following recommended configuration:

```bash
# Set variables
RESOURCE_GROUP="netbird-rg"
CLUSTER_NAME="netbird-aks"
LOCATION="eastus"
NODE_COUNT=3
NODE_SIZE="Standard_D2s_v3"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create AKS cluster
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $NODE_SIZE \
  --enable-managed-identity \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --network-plugin azure \
  --network-policy azure

# Get cluster credentials
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

### 2. Configure Microsoft Entra App Registration

#### Step 2.1: Create App Registration

1. Go to the [Azure Portal](https://portal.azure.com)
2. Navigate to **Microsoft Entra ID** → **App registrations** → **New registration**
3. Configure the registration:
   - **Name**: `Netbird`
   - **Supported account types**: Choose based on your requirements (typically "Single tenant")
   - **Redirect URI**: Leave empty for now (we'll add it after deployment)

4. Click **Register**

#### Step 2.2: Note Important Values

After registration, copy these values (you'll need them later):

- **Application (client) ID**: Found on the Overview page
- **Directory (tenant) ID**: Found on the Overview page

#### Step 2.3: Create Client Secret

1. Navigate to **Certificates & secrets**
2. Click **New client secret**
3. Add a description (e.g., "Netbird Production Secret")
4. Choose expiration (recommendation: 24 months with calendar reminder to rotate)
5. Click **Add**
6. **IMPORTANT**: Copy the secret value immediately (it won't be shown again)

#### Step 2.4: Configure API Permissions

1. Navigate to **API permissions**
2. Click **Add a permission** → **Microsoft Graph** → **Delegated permissions**
3. Add the following permissions:
   - `openid`
   - `profile`
   - `email`
   - `offline_access`
   - `User.Read`
4. Click **Add permissions**
5. Click **Grant admin consent** (requires admin privileges)

#### Step 2.5: Configure Redirect URI

**NOTE**: This step must be done after you know your domain/ingress URL.

1. Navigate to **Authentication**
2. Click **Add a platform** → **Web**
3. Add the redirect URI: `https://YOUR_DOMAIN.com/auth/callback`
   - Replace `YOUR_DOMAIN.com` with your actual domain
4. Under **Implicit grant and hybrid flows**, ensure nothing is checked
5. Click **Configure**

#### Step 2.6: Expose an API (Optional but Recommended)

1. Navigate to **Expose an API**
2. Click **Set** next to Application ID URI
3. Accept the default (`api://YOUR_CLIENT_ID`) or customize
4. Click **Save**

### 3. Configure DNS

You need a domain name pointing to your AKS ingress controller's public IP.

#### Option A: Using Azure DNS

```bash
# Create a DNS zone
az network dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name YOUR_DOMAIN.com

# Note the nameservers and update your domain registrar
az network dns zone show \
  --resource-group $RESOURCE_GROUP \
  --name YOUR_DOMAIN.com \
  --query nameServers
```

#### Option B: Using External DNS Provider

Point an A record to your ingress controller's public IP (obtained after ingress installation).

## Deployment Steps

### 1. Install Prerequisites

#### Install NGINX Ingress Controller

```bash
# Add NGINX ingress Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

# Get the external IP (may take a few minutes)
kubectl get service ingress-nginx-controller -n ingress-nginx -w
```

Note the `EXTERNAL-IP` for DNS configuration.

#### Install cert-manager (for TLS certificates)

```bash
# Add cert-manager Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Create ClusterIssuer for Let's Encrypt
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: YOUR_EMAIL@example.com  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

### 2. Configure Values

1. Copy the example values file:
   ```bash
   cp values-aks-entra.yaml my-values.yaml
   ```

2. Edit `my-values.yaml` and replace all placeholders:
   - `YOUR_DOMAIN.com` → Your actual domain (e.g., `netbird.example.com`)
   - `YOUR_TENANT_ID` → Azure Directory (tenant) ID from Step 2.2
   - `YOUR_CLIENT_ID` → Azure Application (client) ID from Step 2.2

3. Review and adjust resource limits based on your expected load

### 3. Create Secrets

Create Kubernetes secrets for sensitive data:

```bash
# Create netbird namespace
kubectl create namespace netbird

# Create Entra authentication secret
kubectl create secret generic netbird-auth-entra \
  --from-literal=client-id='YOUR_CLIENT_ID' \
  --from-literal=client-secret='YOUR_CLIENT_SECRET' \
  -n netbird

# Generate and create relay secret
RELAY_SECRET=$(openssl rand -base64 32)
kubectl create secret generic netbird-relay-secret \
  --from-literal=relay-secret="$RELAY_SECRET" \
  -n netbird

# Verify secrets were created
kubectl get secrets -n netbird
```

**SECURITY NOTE**: Store these values securely (e.g., Azure Key Vault). For production, consider using:
- Azure Key Vault with Secrets Store CSI Driver
- Sealed Secrets
- External Secrets Operator

### 4. Deploy Netbird

```bash
# Add Netbird Helm repository
helm repo add netbird https://netbirdio.github.io/helms
helm repo update

# Install Netbird using your customized values
helm install netbird netbird/netbird \
  --namespace netbird \
  --values my-values.yaml \
  --wait

# Check deployment status
kubectl get pods -n netbird
kubectl get ingress -n netbird
```

## Post-Deployment Configuration

### 1. Update DNS Records

Point your domain to the ingress controller's external IP:

```bash
# Get the ingress external IP
INGRESS_IP=$(kubectl get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Point YOUR_DOMAIN.com to $INGRESS_IP"
```

Create an A record in your DNS provider:
- **Name**: `@` or `netbird` (depending on subdomain choice)
- **Type**: `A`
- **Value**: `$INGRESS_IP`
- **TTL**: `300` (5 minutes)

### 2. Update Azure App Registration Redirect URI

If you haven't already:

1. Go to Azure Portal → Microsoft Entra ID → App registrations → Netbird
2. Navigate to **Authentication**
3. Ensure the redirect URI is set to: `https://YOUR_DOMAIN.com/auth/callback`

### 3. Verify TLS Certificate

Wait for cert-manager to issue the TLS certificate (usually 1-2 minutes):

```bash
kubectl get certificate -n netbird
kubectl describe certificate netbird-tls -n netbird
```

## Verification

### 1. Check All Components

```bash
# Check all pods are running
kubectl get pods -n netbird

# Expected output: All pods in Running state
# - netbird-dashboard-xxxxx
# - netbird-management-xxxxx
# - netbird-signal-xxxxx
# - netbird-relay-xxxxx

# Check services
kubectl get svc -n netbird

# Check ingress
kubectl get ingress -n netbird
```

### 2. Access the Dashboard

1. Open your browser and navigate to `https://YOUR_DOMAIN.com`
2. You should see the Netbird dashboard
3. Click on the login button
4. You should be redirected to Microsoft Entra ID login page
5. After successful authentication, you should be redirected back to the dashboard

### 3. Test API Endpoints

```bash
# Test management API health
curl -k https://YOUR_DOMAIN.com/api/health

# Expected: {"status":"ok"}
```

## Security Recommendations

### 1. Network Security

- **Enable Network Policies**: Restrict pod-to-pod communication
  ```bash
  # Example NetworkPolicy for management service
  kubectl apply -f - <<EOF
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: management-netpol
    namespace: netbird
  spec:
    podSelector:
      matchLabels:
        app.kubernetes.io/component: management
    policyTypes:
    - Ingress
    - Egress
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: ingress-nginx
    egress:
    - to:
      - namespaceSelector: {}
  EOF
  ```

- **Use Private AKS Cluster**: For enhanced security, consider using a private AKS cluster with private endpoint

### 2. Secrets Management

- **Use Azure Key Vault**: Integrate with Azure Key Vault for secrets management
  ```bash
  # Install Secrets Store CSI Driver
  helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
  helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure \
    --namespace kube-system
  ```

- **Rotate Secrets Regularly**: Set up a process to rotate client secrets and relay secrets

### 3. Access Control

- **Enable RBAC**: Ensure Kubernetes RBAC is enabled (default in AKS)
- **Use Azure RBAC for AKS**: Integrate AKS with Azure RBAC for unified access control
- **Least Privilege**: Grant minimum required permissions to service accounts

### 4. Monitoring and Logging

- **Enable Container Insights**: Monitor cluster health and performance
- **Set up Alerts**: Create alerts for pod failures, high resource usage, etc.
- **Enable Audit Logs**: Track who accessed what resources

### 5. TLS and Certificates

- **Use Strong TLS Configuration**: Ensure TLS 1.2+ is enforced
- **Certificate Rotation**: cert-manager automatically rotates certificates before expiration
- **Monitor Certificate Expiry**: Set up alerts for certificate expiration

### 6. Pod Security

- **Use Pod Security Standards**: Apply restricted pod security standards
- **Run as Non-Root**: Configure containers to run as non-root users
- **Read-Only Root Filesystem**: Where possible, use read-only root filesystems

### 7. Regular Updates

- **Keep Components Updated**: Regularly update Netbird, Kubernetes, and all dependencies
- **Security Patches**: Subscribe to security advisories and apply patches promptly
- **Test Updates**: Always test updates in a non-production environment first

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n netbird

# Check pod logs
kubectl logs <pod-name> -n netbird

# Describe pod for events
kubectl describe pod <pod-name> -n netbird
```

### Ingress Not Working

```bash
# Check ingress status
kubectl describe ingress -n netbird

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Verify cert-manager issued certificate
kubectl get certificate -n netbird
kubectl describe certificate netbird-tls -n netbird
```

### Authentication Errors

1. **Verify Azure App Registration**:
   - Check client ID and tenant ID are correct
   - Ensure client secret hasn't expired
   - Verify redirect URI matches exactly

2. **Check Secrets**:
   ```bash
   # Verify secrets exist
   kubectl get secret netbird-auth-entra -n netbird -o yaml
   ```

3. **Review Management Logs**:
   ```bash
   kubectl logs -l app.kubernetes.io/component=management -n netbird
   ```

### Database Issues

```bash
# Check persistent volume claims
kubectl get pvc -n netbird

# Check persistent volume
kubectl describe pvc <pvc-name> -n netbird

# If using SQLite, check management pod's mounted volume
kubectl exec -it <management-pod> -n netbird -- ls -la /var/lib/netbird
```

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| "failed to get OIDC configuration" | Wrong tenant ID or network issue | Verify tenant ID, check pod network connectivity |
| "invalid client secret" | Wrong or expired secret | Recreate secret with correct value |
| "redirect_uri mismatch" | Azure App Registration mismatch | Update redirect URI in Azure Portal |
| "ImagePullBackOff" | Cannot pull Docker image | Check image name/tag, network connectivity |
| "CrashLoopBackOff" | Application error | Check pod logs for specific error |

## Maintenance

### Backup

Backup the management service's persistent volume regularly:

```bash
# Using Velero or Azure Backup
# Example with Velero:
velero backup create netbird-backup --include-namespaces netbird
```

### Scaling

```bash
# Scale deployments manually
kubectl scale deployment netbird-management --replicas=3 -n netbird

# Or update values.yaml and upgrade
helm upgrade netbird netbird/netbird \
  --namespace netbird \
  --values my-values.yaml
```

### Upgrading Netbird

```bash
# Update Helm repository
helm repo update

# Check for new versions
helm search repo netbird/netbird --versions

# Upgrade to latest version
helm upgrade netbird netbird/netbird \
  --namespace netbird \
  --values my-values.yaml

# Or upgrade to specific version
helm upgrade netbird netbird/netbird \
  --namespace netbird \
  --values my-values.yaml \
  --version 1.10.0
```

### Monitoring

Set up monitoring using Prometheus and Grafana:

```bash
# Enable metrics in values.yaml
metrics:
  serviceMonitor:
    enabled: true

# If using kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

## Support and Resources

- **Netbird Documentation**: https://docs.netbird.io
- **Netbird GitHub**: https://github.com/netbirdio/netbird
- **Helm Chart Repository**: https://github.com/netbirdio/helms
- **Community Slack**: https://netbird.io/community
- **Microsoft Entra Documentation**: https://learn.microsoft.com/entra

## Contributing

If you find issues with this deployment guide or have improvements, please contribute to the repository.

## License

This deployment configuration follows the Netbird licensing terms. See the main Netbird repository for details.

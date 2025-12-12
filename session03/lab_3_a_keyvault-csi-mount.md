# Lab 3a: Key Vault CSI Mount
<img width="1412" height="649" alt="ZIMAGE" src="https://github.com/user-attachments/assets/b7de30ea-9e18-4766-b9f0-42e6d3dc6734" />

## Objective
Mount Azure Key Vault secrets into AKS pods using the **Key Vault CSI Driver** with full parameterization, comprehensive testing, and production-grade security practices. This lab demonstrates secure secret management without storing credentials in code, container images, or Kubernetes manifests.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure subscription quota for AKS and Key Vault

---

## 1. Set Lab Parameters

```bash
# Azure resource configuration
RESOURCE_GROUP="rg-aks-keyvault-csi-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-keyvault-csi"

# AKS cluster configuration
NODE_COUNT=3
NODE_VM_SIZE="Standard_DS2_v2"

# Key Vault name must be globally unique (3-24 chars, alphanumeric + hyphens)
KEY_VAULT_NAME="kv-aks-$(openssl rand -hex 3)"

# Retrieve Azure tenant ID
TENANT_ID=$(az account show \
  --query tenantId \
  --output tsv)

# Retrieve subscription ID
SUBSCRIPTION_ID=$(az account show \
  --query id \
  --output tsv)

# Display all configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Node Count: $NODE_COUNT"
echo "Key Vault Name: $KEY_VAULT_NAME"
echo "Tenant ID: $TENANT_ID"
echo "Subscription ID: $SUBSCRIPTION_ID"
echo "========================="
```

---

## 2. Create Resource Group and AKS Cluster

```bash
# Create resource group
RG_RESULT=$(az group create \
  --name $RESOURCE_GROUP `# Resource group name` \
  --location $LOCATION `# Azure region` \
  --query 'properties.provisioningState' \
  --output tsv)

echo "Resource Group: $RG_RESULT"

# Create AKS cluster with managed identity (5-10 minutes)
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_VM_SIZE `# Node VM size` \
  --enable-managed-identity `# Use system-assigned managed identity` \
  --generate-ssh-keys `# Auto-generate SSH keys` \
  --query 'provisioningState' \
  --output tsv)

echo "AKS Cluster: $CLUSTER_RESULT"

# Get AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP `# Source resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --overwrite-existing `# Replace existing kubeconfig entry` \
  --output none

# Verify cluster connectivity
echo ""
echo "Current kubectl context: $(kubectl config current-context)"
echo ""
echo "Cluster nodes:"
kubectl get nodes
```

---

## 3. Create Azure Key Vault

```bash
# Create Key Vault for storing secrets
KV_RESULT=$(az keyvault create \
  --name $KEY_VAULT_NAME `# Globally unique Key Vault name` \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --location $LOCATION `# Azure region` \
  --enable-rbac-authorization false `# Use access policies (not Azure RBAC)` \
  --query 'properties.provisioningState' \
  --output tsv)

echo "Key Vault: $KV_RESULT"

# Display Key Vault details
KV_URI=$(az keyvault show \
  --name $KEY_VAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query 'properties.vaultUri' \
  --output tsv)

echo "Key Vault URI: $KV_URI"
```

---

## 4. Add Secrets to Key Vault

```bash
# Store sample secrets in Key Vault
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME `# Target Key Vault` \
  --name database-password `# Secret name` \
  --value "SuperSecretP@ss1" `# Secret value` \
  --output none

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name api-key \
  --value "abc1234567890" \
  --output none

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name connection-string \
  --value "Server=sql;Database=db;User=admin;Password=xyz;" \
  --output none

# Verify all secrets
az keyvault secret list \
  --vault-name $KEY_VAULT_NAME \
  --query '[].{Name:name, Enabled:attributes.enabled, Created:attributes.created}' \
  --output table
```

---

## 5. Enable Key Vault CSI Driver Add-on

```bash
# Install the AKS add-on that mounts Key Vault secrets into pods
ADDON_RESULT=$(az aks enable-addons \
  --addons azure-keyvault-secrets-provider `# CSI driver add-on name` \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --query 'provisioningState' \
  --output tsv)

echo "CSI Driver: $ADDON_RESULT"

# Wait for driver pods to be ready
kubectl wait --for=condition=ready \
  --timeout=120s \
  pod -n kube-system \
  -l app.kubernetes.io/name=secrets-store-csi-driver 2>/dev/null || \
  echo "CSI driver pods starting..."

# Verify CSI driver installation
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=secrets-store-csi-driver

kubectl get pods -n kube-system \
  -l app=secrets-store-provider-azure

# Verify CSI driver is registered
kubectl get csidrivers
```

---

## 6. Retrieve AKS Managed Identity

```bash
# Get the kubelet identity used by AKS to access Azure resources
IDENTITY_CLIENT_ID=$(az aks show \
  --resource-group $RESOURCE_GROUP `# AKS resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --query 'identityProfile.kubeletidentity.clientId' `# Kubelet identity client ID` \
  --output tsv)

echo "Kubelet Identity Client ID: $IDENTITY_CLIENT_ID"

# Get the identity's object ID (for role assignments)
IDENTITY_OBJECT_ID=$(az ad sp show \
  --id $IDENTITY_CLIENT_ID `# Service principal ID` \
  --query id `# Object ID` \
  --output tsv)

echo "Identity Object ID: $IDENTITY_OBJECT_ID"

# Display identity details
az identity show \
  --ids $(az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query 'identityProfile.kubeletidentity.resourceId' \
    --output tsv) \
  --query '{Name:name, Type:type, PrincipalId:principalId, ClientId:clientId}' \
  --output table
```

---

## 7. Grant Key Vault Access to AKS Managed Identity

```bash
# Grant AKS kubelet identity permission to read Key Vault secrets
# Using Access Policies (recommended when RBAC authorization is disabled)
az keyvault set-policy \
  --name $KEY_VAULT_NAME `# Target Key Vault` \
  --object-id $IDENTITY_OBJECT_ID `# AKS identity object ID` \
  --secret-permissions get list `# Read-only secret access` \
  --output none

# Option 2: Using Azure RBAC (if --enable-rbac-authorization true was set)
# Uncomment if using RBAC authorization:
# KV_SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KEY_VAULT_NAME"
# az role assignment create \
#   --role "Key Vault Secrets User" \
#   --assignee $IDENTITY_CLIENT_ID \
#   --scope $KV_SCOPE \
#   --output none

# Verify access policy
echo ""
# Verify access policy
az keyvault showup $RESOURCE_GROUP \
  --query 'properties.accessPolicies[].{ObjectId:objectId, Permissions:permissions}' \
  --output table
```

---

## 8. Create SecretProviderClass

```bash
# Define which Key Vault secrets to mount into pods
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kv-csi
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"  # Not using AAD Pod Identity
    useVMManagedIdentity: "true"  # Use AKS kubelet managed identity
    userAssignedIdentityID: "$IDENTITY_CLIENT_ID"  # Kubelet identity client ID
    keyvaultName: "$KEY_VAULT_NAME"  # Target Key Vault name
    cloudName: ""  # Use Azure Public Cloud
    tenantId: "$TENANT_ID"  # Azure AD tenant ID
    objects: |
      array:
        - |
          objectName: database-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: api-key
          objectType: secret
          objectVersion: ""
        - |
          objectName: connection-string
          objectType: secret
          objectVersion: ""
EOF

# Verify SecretProviderClass
kubectl get secretproviderclass azure-kv-csi
```

---

## 9. Deploy Pod with CSI Volume Mount

```bash
# Deploy pod that mounts Key Vault secrets as files
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-secrets-demo
  namespace: default
  labels:
    app: secrets-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: kv-secrets
      mountPath: "/mnt/secrets"
      readOnly: true
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
  volumes:
  - name: kv-secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-kv-csi"
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/app-secrets-demo
```

---

## 10. Verify Secret Mounting

```bash
# Check pod status
kubectl get pod app-secrets-demo

# List mounted secret files
kubectl exec app-secrets-demo -- ls -la /mnt/secrets

# Display secret values (be careful in production!)
kubectl exec app-secrets-demo -- cat /mnt/secrets/database-password
kubectl exec app-secrets-demo -- cat /mnt/secrets/api-key
kubectl exec app-secrets-demo -- cat /mnt/secrets/connection-string

# Verify secrets are read-only
kubectl exec app-secrets-demo -- \
  sh -c 'echo "test" > /mnt/secrets/test 2>&1' || \
  echo "✓ Secrets are read-only"
```

---

## 11. Sync Secrets to Kubernetes Secrets (Optional)

```bash
# Create SecretProviderClass that syncs Key Vault secrets to K8s Secrets
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kv-sync
  namespace: default
spec:
  provider: azure
  # Sync Key Vault secrets to Kubernetes Secret objects
  secretObjects:
  - secretName: synced-secrets
    type: Opaque
    data:
    - objectName: database-password
      key: database-password
    - objectName: api-key
      key: api-key
    - objectName: connection-string
      key: connection-string
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "$IDENTITY_CLIENT_ID"
    keyvaultName: "$KEY_VAULT_NAME"
    tenantId: "$TENANT_ID"
    objects: |
      array:
        - |
          objectName: database-password
          objectType: secret
        - |
          objectName: api-key
          objectType: secret
        - |
          objectName: connection-string
          objectType: secret
EOF
```

---

## 12. Deploy Pod with Synced Secrets

```bash
# Deploy pod that uses synced Kubernetes Secrets
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-env-secrets
  namespace: default
  labels:
    app: env-secrets-demo
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
    env:
    # Inject secrets as environment variables
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: synced-secrets
          key: database-password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: synced-secrets
          key: api-key
    - name: CONNECTION_STRING
      valueFrom:
        secretKeyRef:
          name: synced-secrets
          key: connection-string
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-kv-sync"
EOF

# Wait for pod and secret sync
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/app-with-env-secrets

# Verify Kubernetes Secret was created
kubectl get secret synced-secrets
kubectl describe secret synced-secrets

# View environment variables in pod
kubectl exec app-with-env-secrets -- env | grep -E "DATABASE|API|CONNECTION"
```

---

## 13. Deploy with Deployment (Production Pattern)

```bash
# Create deployment with multiple replicas using secrets
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: synced-secrets
              key: database-password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: synced-secrets
              key: api-key
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
      volumes:
      - name: secrets-store
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "azure-kv-sync"
EOF

# Wait for deployment to be ready
kubectl rollout status deployment/secure-app --timeout=120s

# List all pods
kubectl get pods -l app=secure-app

# Test secret access from one pod
POD_NAME=$(kubectl get pod -l app=secure-app -o jsonpath='{.items[0].metadata.name}')
echo "Testing pod: $POD_NAME"
kubectl exec $POD_NAME -- cat /mnt/secrets/database-password
```

---

## 14. Test Secret Rotation

```bash
# Update secret in Key Vault to test rotation
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name database-password \
  --value "NewRotatedP@ssw0rd456" \
  --output none

# Restart deployment to pick up new secret
kubectl rollout restart deployment/secure-app
kubectl rollout status deployment/secure-app --timeout=120s

# Verify new secret value
POD_NAME=$(kubectl get pod -l app=secure-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- cat /mnt/secrets/database-password
```

---

## 15. Monitor and Troubleshoot CSI Driver

```bash
# Check CSI driver logs
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=secrets-store-csi-driver \
  --tail=20

# Check Azure provider logs
kubectl logs -n kube-system \
  -l app=secrets-store-provider-azure \
  --tail=20

# View CSI nodes
kubectl get csinodes

# View all CSI drivers
kubectl get csidrivers

# Check SecretProviderClass status
kubectl get secretproviderclass -A

# View secret provider class pod status
kubectl get secretproviderclasspodstatus -A 2>/dev/null || \
  echo "No active SecretProviderClassPodStatus resources"
```

---

## Expected Results

✅ Key Vault CSI driver successfully installed  
✅ Azure Key Vault created with secrets  
✅ AKS managed identity granted access to Key Vault  
✅ SecretProviderClass configured correctly  
✅ Secrets mounted as files in pod volumes (`/mnt/secrets/`)  
✅ Secrets synced to Kubernetes Secret objects  
✅ Environment variables populated from synced secrets  
✅ Deployment with multiple replicas accessing secrets  
✅ Secret rotation working (requires pod restart)  
✅ Read-only secret mounts enforced  

---

## How This Connects to Secure Secret Management

This lab demonstrates **production-grade secret management** for AKS:

### Security Architecture:

```
┌─────────────────────────────────────────────┐
│            AKS Cluster                      │
│                                             │
│  ┌────────────────┐                        │
│  │     Pod        │                        │
│  │  ┌──────────┐  │                        │
│  │  │ CSI      │  │                        │
│  │  │ Volume   │  │                        │
│  │  │ Mount    │  │                        │
│  │  └────┬─────┘  │                        │
│  └───────┼────────┘                        │
│          │                                  │
│          ▼                                  │
│  ┌────────────────┐                        │
│  │  CSI Driver    │◄────Managed Identity   │
│  │  (DaemonSet)   │                        │
│  └───────┬────────┘                        │
│          │                                  │
└──────────┼──────────────────────────────────┘
           │
           │ HTTPS (TLS 1.2+)
           │ Azure AD Authentication
           ▼
┌─────────────────────────────┐
│   Azure Key Vault           │
│                             │
│   ┌─────────────────────┐   │
│   │ Secrets             │   │
│   │ - database-password │   │
│   │ - api-key           │   │
│   │ - connection-string │   │
│   └─────────────────────┘   │
│                             │
│   Access Policies:          │
│   ✓ Managed Identity       │
│   ✓ Get/List Secrets       │
└─────────────────────────────┘
```

### Enterprise Use Cases:

| Pattern | Implementation | Benefit |
|---------|---------------|---------|
| **External Secret Store** | Key Vault CSI Driver | Secrets never stored in cluster etcd |
| **Workload Identity** | Managed Identity | No credentials in code or config |
| **Secret Rotation** | Pod restart triggers refresh | Zero-downtime secret updates |
| **K8s Secret Sync** | secretObjects configuration | Compatible with existing apps using env vars |
| **Read-Only Mounts** | CSI volume attributes | Prevents pod from modifying secrets |
| **Least Privilege** | Key Vault access policies | Granular permission control |

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete all deployed workloads and configurations
kubectl delete deployment secure-app --ignore-not-found
kubectl delete pod app-with-env-secrets --ignore-not-found
kubectl delete pod app-secrets-demo --ignore-not-found
kubectl delete secretproviderclass azure-kv-csi azure-kv-sync --ignore-not-found
kubectl delete secret synced-secrets --ignore-not-found
```

### Option 2: Delete Key Vault and Disable CSI Driver
```bash
# Delete Key Vault (moves to soft-delete state)
az keyvault delete \
  --name $KEY_VAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --output none

# Purge Key Vault permanently
az keyvault purge \
  --name $KEY_VAULT_NAME \
  --output none

# Disable CSI driver add-on (optional - keeps cluster)
az aks disable-addons \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --addons azure-keyvault-secrets-provider \
  --output none
```

### Option 3: Delete Entire Resource Group (Complete Cleanup)
```bash
# Delete entire resource group (removes all resources)
echo "Deleting resource group: $RESOURCE_GROUP..."
DELETE_RESULT=$(az group delete \
  --name $RESOURCE_GROUP `# Target resource group` \
  --yes `# Skip confirmation` \
  --no-wait `# Run asynchronously` \
  --output tsv)

echo "Delete operation initiated: $DELETE_RESULT"

# Verify deletion status (run after a few minutes)
echo ""
echo "To verify deletion status, run:"
echo "az group show --name $RESOURCE_GROUP --query properties.provisioningState"
```

---

## Key Takeaways

- **CSI Driver** mounts Key Vault secrets as read-only volumes in pods
- **Managed Identity** provides passwordless authentication to Key Vault
- **SecretProviderClass** defines which secrets to mount and sync
- **Secret Sync** creates Kubernetes Secrets for env var injection
- **No credentials in code** - secrets retrieved dynamically at runtime
- **Auto-rotation** available through pod restarts (not live updates)
- **Multiple mounts** - same SecretProviderClass used by many pods
- **Access policies** control which identities can read secrets
- **Soft delete** protects against accidental Key Vault deletion
- **Production-ready** pattern for enterprise secret management

---

## Security Best Practices

| Practice | Implementation | Rationale |
|----------|---------------|-----------|
| **Use Managed Identity** | Enable on AKS cluster | No credentials to manage or rotate |
| **Least privilege access** | Grant only get/list permissions | Limit blast radius of compromise |
| **Enable Key Vault firewall** | Restrict to AKS subnet | Network-level protection |
| **Use separate Key Vaults** | Per environment (dev/staging/prod) | Isolation and access control |
| **Enable audit logging** | Key Vault diagnostic logs | Track secret access |
| **Rotate secrets regularly** | Automated rotation policy | Reduce exposure window |
| **Read-only mounts** | CSI volume readOnly: true | Prevent pod tampering |
| **Avoid secret logging** | Never log secret values | Prevent exposure in logs |

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| **Pod stuck ContainerCreating** | CSI driver can't auth to Key Vault | Check managed identity has access policy |
| **Secrets not mounting** | Incorrect SecretProviderClass config | Verify objectName matches Key Vault secret name |
| **Empty secret files** | Identity lacks permissions | Grant get/list secret permissions |
| **Synced K8s secret not created** | secretObjects missing or pod not started | Check SecretProviderClass has secretObjects defined |
| **403 Forbidden errors** | Access policy not configured | Run `az keyvault set-policy` command |
| **Rotation not working** | Secrets cached until pod restart | Restart deployment: `kubectl rollout restart` |
| **CSI driver pods crashing** | Resource constraints | Check CSI driver pod logs and resource requests |

---

## Additional Resources

- [Azure Key Vault Provider for Secrets Store CSI Driver](https://azure.github.io/secrets-store-csi-driver-provider-azure/)
- [Use Azure Key Vault with AKS](https://learn.microsoft.com/azure/aks/csi-secrets-store-driver)
- [Key Vault Best Practices](https://learn.microsoft.com/azure/key-vault/general/best-practices)
- [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/)

---

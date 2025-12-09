# Lab 3a: Key Vault CSI Mount

## Objective
Mount Key Vault secrets into pods using the CSI driver.

## Prerequisites
- AKS cluster running
- Azure Key Vault instance
- Azure CLI and `kubectl` configured
- Managed identity enabled on AKS

## Steps

### 1. Create Azure Key Vault
```bash
# Set variables
RESOURCE_GROUP="<resource-group>"
KEY_VAULT_NAME="akskv$(openssl rand -hex 4)"
LOCATION="eastus"

# Create Key Vault
az keyvault create \
  --name $KEY_VAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enable-rbac-authorization false
```

### 2. Add Secrets to Key Vault
```bash
# Create sample secrets
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name database-password \
  --value "SuperSecretP@ssw0rd123"

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name api-key \
  --value "sk-1234567890abcdef"

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name connection-string \
  --value "Server=myserver.database.windows.net;Database=mydb;User=admin;Password=P@ssw0rd;"

# Verify secrets
az keyvault secret list --vault-name $KEY_VAULT_NAME -o table
```

### 3. Enable Key Vault CSI Driver Add-on
```bash
az aks enable-addons \
  --resource-group $RESOURCE_GROUP \
  --name <cluster-name> \
  --addons azure-keyvault-secrets-provider
```

Verify installation:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=secrets-store-provider-azure
```

### 4. Get AKS Managed Identity
```bash
# Get the kubelet identity (user-assigned managed identity)
IDENTITY_CLIENT_ID=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name <cluster-name> \
  --query "identityProfile.kubeletidentity.clientId" -o tsv)

echo "Kubelet Identity Client ID: $IDENTITY_CLIENT_ID"

# Get the identity's object ID
IDENTITY_OBJECT_ID=$(az ad sp show \
  --id $IDENTITY_CLIENT_ID \
  --query id -o tsv)

echo "Identity Object ID: $IDENTITY_OBJECT_ID"
```

### 5. Grant Key Vault Access to Managed Identity
```bash
# Assign Key Vault Secrets User role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $IDENTITY_CLIENT_ID \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KEY_VAULT_NAME"

# Alternative: Use access policies (if RBAC not enabled)
az keyvault set-policy \
  --name $KEY_VAULT_NAME \
  --object-id $IDENTITY_OBJECT_ID \
  --secret-permissions get list
```

### 6. Create SecretProviderClass
Create `secretproviderclass.yaml`:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "<IDENTITY_CLIENT_ID>"  # Replace with actual value
    keyvaultName: "<KEY_VAULT_NAME>"                # Replace with actual value
    cloudName: ""
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
    tenantId: "<TENANT_ID>"  # Get with: az account show --query tenantId -o tsv
```

Apply with actual values:
```bash
TENANT_ID=$(az account show --query tenantId -o tsv)

cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "$IDENTITY_CLIENT_ID"
    keyvaultName: "$KEY_VAULT_NAME"
    cloudName: ""
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
    tenantId: "$TENANT_ID"
EOF
```

### 7. Deploy Pod with CSI Volume Mount
Create `pod-with-secrets.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
    env:
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-keyvault-secrets"
```

Apply the pod:
```bash
kubectl apply -f pod-with-secrets.yaml
```

### 8. Verify Secret Mounting
```bash
# Check pod status
kubectl get pod app-with-secrets

# Verify secrets are mounted
kubectl exec app-with-secrets -- ls -la /mnt/secrets

# Read secret values
kubectl exec app-with-secrets -- cat /mnt/secrets/database-password
kubectl exec app-with-secrets -- cat /mnt/secrets/api-key
kubectl exec app-with-secrets -- cat /mnt/secrets/connection-string
```

### 9. Sync Secrets to Kubernetes Secrets (Optional)
Modify SecretProviderClass to sync to Kubernetes secrets:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets-sync
  namespace: default
spec:
  provider: azure
  secretObjects:
  - secretName: app-secrets
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
    tenantId: "$TENANT_ID"
```

Apply and deploy pod:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets-sync
  namespace: default
spec:
  provider: azure
  secretObjects:
  - secretName: app-secrets
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
    tenantId: "$TENANT_ID"
EOF
```

Update pod to use synced secret:
```bash
kubectl delete pod app-with-secrets

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-synced-secrets
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
    env:
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-keyvault-secrets-sync"
EOF
```

Verify Kubernetes secret was created:
```bash
kubectl get secret app-secrets
kubectl describe secret app-secrets

# View environment variables in pod
kubectl exec app-with-synced-secrets -- env | grep -E "DATABASE|API"
```

### 10. Use with Deployment
Create `deployment-with-secrets.yaml`:

```yaml
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
              name: app-secrets
              key: database-password
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
            secretProviderClass: "azure-keyvault-secrets-sync"
```

```bash
kubectl apply -f deployment-with-secrets.yaml
kubectl get pods -l app=secure-app
```

### 11. Enable Auto-Rotation
Secrets are auto-rotated when the pod restarts. Configure rotation:

```bash
# View CSI driver configuration
kubectl get pod -n kube-system -l app.kubernetes.io/name=secrets-store-csi-driver -o yaml | grep -A 5 rotation

# Secrets rotate on pod restart or based on polling interval
# Default: Checked every 2 minutes
```

Test rotation:
```bash
# Update secret in Key Vault
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name database-password \
  --value "NewRotatedP@ssw0rd456"

# Restart pod to get new secret
kubectl rollout restart deployment secure-app

# Wait for pods to restart
kubectl rollout status deployment secure-app

# Verify new secret
kubectl exec -it $(kubectl get pod -l app=secure-app -o jsonpath='{.items[0].metadata.name}') -- cat /mnt/secrets/database-password
```

### 12. Monitor CSI Driver
```bash
# Check driver logs
kubectl logs -n kube-system -l app.kubernetes.io/name=secrets-store-csi-driver --tail=50

# Check provider logs
kubectl logs -n kube-system -l app=secrets-store-provider-azure --tail=50

# View CSI driver metrics
kubectl get csinodes
kubectl get csidrivers
```

## Expected Results
- Key Vault CSI driver successfully installed
- Secrets mounted as files in pod volumes
- Secrets synced to Kubernetes Secret objects
- Environment variables populated from secrets
- Secrets auto-rotate on pod restart
- Multiple pods can access same secrets
- Managed identity authenticates to Key Vault

## Cleanup
```bash
kubectl delete deployment secure-app
kubectl delete pod app-with-synced-secrets --ignore-not-found
kubectl delete secretproviderclass azure-keyvault-secrets azure-keyvault-secrets-sync
kubectl delete secret app-secrets --ignore-not-found

# Delete Key Vault
az keyvault delete --name $KEY_VAULT_NAME --resource-group $RESOURCE_GROUP
az keyvault purge --name $KEY_VAULT_NAME

# Disable add-on (optional)
az aks disable-addons \
  --resource-group $RESOURCE_GROUP \
  --name <cluster-name> \
  --addons azure-keyvault-secrets-provider
```

## Key Takeaways
- **CSI driver** mounts Key Vault secrets as volumes in pods
- **Managed Identity** provides secure authentication without credentials
- **SecretProviderClass** defines which secrets to mount
- **Auto-rotation** ensures secrets stay current
- **Sync to Kubernetes Secrets** enables env var injection
- Secrets are mounted per-pod, not shared across pods
- No secrets stored in container images or YAML files

## Architecture

```
┌─────────────┐
│   Pod       │
│  ┌────────┐ │      ┌──────────────────┐
│  │ CSI    │─┼──────┤ CSI Driver       │
│  │ Volume │ │      │ (DaemonSet)      │
│  └────────┘ │      └─────────┬────────┘
└─────────────┘                │
                               │ Managed Identity
                               ▼
                    ┌──────────────────┐
                    │  Azure Key Vault │
                    │  - Secrets       │
                    │  - Keys          │
                    │  - Certificates  │
                    └──────────────────┘
```

## Troubleshooting
- **Pod stuck in ContainerCreating**: Check CSI driver logs for auth errors
- **Secrets not mounting**: Verify managed identity has Key Vault access
- **Empty secret files**: Check SecretProviderClass objectName matches Key Vault
- **Rotation not working**: Secrets update on pod restart, not live refresh

---

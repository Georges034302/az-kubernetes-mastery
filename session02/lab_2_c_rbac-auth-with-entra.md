# Lab 2c: RBAC Authentication with Entra ID

## Objective
Authenticate to AKS using Entra ID (formerly Azure AD) and test RBAC permissions.

## Prerequisites
- AKS cluster with Entra ID integration enabled
- Azure CLI installed
- User account in Entra ID tenant
- Appropriate permissions to manage cluster access

## Steps

### 1. Enable Entra ID Integration (If Not Already Enabled)
For existing cluster:

```bash
# Enable managed Entra ID integration
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --enable-aad \
  --enable-azure-rbac
```

For new cluster:
```bash
az aks create \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --enable-aad \
  --enable-azure-rbac \
  --node-count 2
```

Verify Entra ID is enabled:
```bash
az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query "aadProfile" -o table
```

### 2. Get AKS Credentials with Entra ID
Clear existing credentials and get new ones:

```bash
# Remove cached credentials
rm -f ~/.kube/config

# Get admin credentials (for setup)
az aks get-credentials \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --admin

# Test access
kubectl get nodes
```

### 3. Get User Credentials (Entra ID Authentication)
Now get credentials that require Entra ID authentication:

```bash
az aks get-credentials \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --overwrite-existing
```

Test authentication (will prompt for browser login):
```bash
kubectl get nodes
```

**Expected:** Browser opens for device code authentication or interactive login.

### 4. Verify Current Identity
Check which Entra ID account is authenticated:

```bash
az account show --query user -o table

kubectl auth whoami
```

### 5. Assign Azure RBAC Roles
Assign the "Azure Kubernetes Service Cluster User Role" to a user:

```bash
# Get user object ID
USER_EMAIL="user@domain.com"
USER_OBJECT_ID=$(az ad user show --id $USER_EMAIL --query id -o tsv)

# Get cluster resource ID
CLUSTER_ID=$(az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query id -o tsv)

# Assign Cluster User role
az role assignment create \
  --assignee $USER_OBJECT_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $CLUSTER_ID
```

### 6. Assign Kubernetes RBAC Permissions
Create a namespace for testing:

```bash
kubectl create namespace dev-team
```

Assign specific Kubernetes permissions using Azure RBAC:

**Option A: Using Azure Built-in Roles**
```bash
# Grant "Azure Kubernetes Service RBAC Reader" at namespace scope
az role assignment create \
  --assignee $USER_OBJECT_ID \
  --role "Azure Kubernetes Service RBAC Reader" \
  --scope "$CLUSTER_ID/namespaces/dev-team"

# Or grant "Azure Kubernetes Service RBAC Writer"
az role assignment create \
  --assignee $USER_OBJECT_ID \
  --role "Azure Kubernetes Service RBAC Writer" \
  --scope "$CLUSTER_ID/namespaces/dev-team"
```

**Option B: Using Kubernetes Native RBAC**
Create `dev-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-role
  namespace: dev-team
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "jobs", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
```

```bash
kubectl apply -f dev-role.yaml
```

Create RoleBinding to Entra ID user:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding
  namespace: dev-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: dev-role
subjects:
- kind: User
  name: $USER_OBJECT_ID  # Entra ID Object ID
  apiGroup: rbac.authorization.k8s.io
```

Apply with substitution:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding
  namespace: dev-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: dev-role
subjects:
- kind: User
  name: $USER_OBJECT_ID
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 7. Test Permissions as User
Switch to user context (if testing with different user):

```bash
# Authenticate as the specific user
az login --username $USER_EMAIL

# Get credentials
az aks get-credentials \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --overwrite-existing
```

Test allowed operations:
```bash
# Should succeed
kubectl get pods -n dev-team
kubectl run test-pod --image=nginx -n dev-team
kubectl get pods -n dev-team

# Should fail (no access to other namespaces)
kubectl get pods -n default
kubectl get pods -n kube-system
```

### 8. Create Entra ID Group for Team Access
```bash
# Create Entra ID group
GROUP_NAME="AKS-DevTeam"
GROUP_ID=$(az ad group create \
  --display-name $GROUP_NAME \
  --mail-nickname $GROUP_NAME \
  --query id -o tsv)

# Add users to group
az ad group member add \
  --group $GROUP_ID \
  --member-id $USER_OBJECT_ID

# Verify group membership
az ad group member list --group $GROUP_ID -o table
```

### 9. Assign RBAC to Entra ID Group
Create group-based RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-group-binding
  namespace: dev-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: dev-role
subjects:
- kind: Group
  name: $GROUP_ID  # Entra ID Group Object ID
  apiGroup: rbac.authorization.k8s.io
```

Apply:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-group-binding
  namespace: dev-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: dev-role
subjects:
- kind: Group
  name: $GROUP_ID
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 10. Test Group-Based Access
Users in the group should now have access:

```bash
kubectl get pods -n dev-team
kubectl auth can-i list pods -n dev-team
kubectl auth can-i create deployments -n dev-team
kubectl auth can-i delete nodes  # Should return 'no'
```

### 11. View Role Assignments
```bash
# View Azure role assignments
az role assignment list \
  --scope $CLUSTER_ID \
  --query "[?principalType=='User'].{User:principalName, Role:roleDefinitionName, Scope:scope}" \
  -o table

# View Kubernetes RBAC
kubectl get rolebindings -n dev-team
kubectl describe rolebinding dev-team-group-binding -n dev-team

kubectl get clusterrolebindings | grep -i azure
```

### 12. Test Permission Boundaries
Create test resources:

```bash
# Should work (namespace scoped)
kubectl create deployment nginx --image=nginx -n dev-team

# Should fail (cluster scoped)
kubectl create namespace test-ns

# Check what you can do
kubectl auth can-i --list -n dev-team
kubectl auth can-i --list --all-namespaces
```

### 13. Conditional Access Integration
If your organization uses Conditional Access policies, AKS will honor them:

```bash
# Authentication will enforce:
# - MFA requirements
# - Device compliance
# - Location restrictions
# - Sign-in risk policies

kubectl get nodes  # May prompt for MFA if policy requires it
```

## Expected Results
- Users authenticate using Entra ID credentials
- Azure RBAC roles control cluster access
- Kubernetes RBAC defines namespace-level permissions
- Group-based access management works correctly
- Users can only access resources within their scope
- `kubectl auth can-i` correctly reflects permissions
- Conditional Access policies are enforced

## Cleanup
```bash
kubectl delete namespace dev-team

# Remove role assignments
az role assignment delete \
  --assignee $USER_OBJECT_ID \
  --scope "$CLUSTER_ID/namespaces/dev-team"

# Delete Entra ID group
az ad group delete --group $GROUP_ID
```

## Key Takeaways
- **Entra ID integration** provides centralized identity management
- **Azure RBAC** controls cluster-level access (get-credentials)
- **Kubernetes RBAC** controls resource access within the cluster
- **Groups** simplify permission management across multiple users
- **Namespace scoping** enables multi-tenancy
- Authentication tokens are cached and auto-refreshed
- Conditional Access policies add security layer
- `kubectl auth can-i` is essential for testing permissions

## Azure RBAC Roles for AKS

| Role | Scope | Permissions |
|------|-------|-------------|
| Cluster Admin | Cluster | Full access including get-credentials |
| Cluster User | Cluster | Can get credentials, requires K8s RBAC for resources |
| RBAC Admin | Namespace/Cluster | Full K8s RBAC permissions |
| RBAC Writer | Namespace/Cluster | Read/write K8s resources |
| RBAC Reader | Namespace/Cluster | Read-only K8s resources |

## Troubleshooting
- **Authentication fails**: Check Entra ID integration with `az aks show`
- **Permissions denied**: Verify both Azure RBAC and Kubernetes RBAC roles
- **Group membership not working**: Allow 5-10 minutes for token refresh
- **Can't get credentials**: Check "Azure Kubernetes Service Cluster User Role"
- **Browser not opening**: Use `az login --use-device-code`

---


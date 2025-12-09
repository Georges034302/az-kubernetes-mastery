# Lab 2c: RBAC Authentication with Microsoft Entra ID
<img width="1536" height="660" alt="ZIMAGE" src="https://github.com/user-attachments/assets/34264934-4dfc-4eba-930c-32d1bb3bcc88" />

## Objective
Build a complete AKS + Microsoft Entra ID RBAC environment from scratch, configure **Azure RBAC** for API server access and **Kubernetes RBAC** for namespace-level authorization, and validate least-privilege access patterns.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Microsoft Entra ID user account with appropriate permissions
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Azure configuration
RESOURCE_GROUP="rg-aks-rbac-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-rbac-demo"

# Cluster configuration
NODE_COUNT=2
NODE_VM_SIZE="Standard_DS2_v2"

# User configuration (UPDATE THIS)
USER_EMAIL="your-email@domain.com"  # Replace with your Entra ID user email

# RBAC configuration
NAMESPACE="rbac-lab"
ROLE_NAME="dev-role"
ROLEBINDING_NAME="dev-rolebinding"
GROUP_NAME="AKS-DevTeam"

# Display configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "User Email: $USER_EMAIL"
echo "Namespace: $NAMESPACE"
echo "========================="
```

---

## 2. Create Resource Group and AKS Cluster with Entra ID

```bash
# Create resource group
RG_RESULT=$(az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --query 'properties.provisioningState' \
  --output tsv)

echo "Resource Group creation: $RG_RESULT"

# Create AKS cluster with Entra ID integration
echo ""
echo "Creating AKS cluster with Entra ID integration (this may take 5-10 minutes)..."
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_VM_SIZE `# Node VM size` \
  --enable-aad `# Enable Entra ID (formerly Azure AD) integration` \
  --enable-azure-rbac `# Enable Azure RBAC for Kubernetes API authorization` \
  --enable-managed-identity `# Use managed identity for cluster identity` \
  --generate-ssh-keys `# Auto-generate SSH keys if not exist` \
  --query 'provisioningState' `# Extract provisioning state only` \
  --output tsv)

echo "AKS cluster creation: $CLUSTER_RESULT"

# Verify Entra ID integration is enabled
echo ""
echo "=== Entra ID Configuration ==="
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query 'aadProfile' \
  --output table
```

Expected output:
- **Managed**: `True`
- **EnableAzureRBAC**: `True`

---

## 3. Configure Azure RBAC for API Server Access

Azure RBAC controls **authentication** and initial authorization to the Kubernetes API server.

### Get User Object ID:

```bash
# Retrieve Entra ID user object ID dynamically
echo "=== Retrieving User Object ID ==="
USER_OBJECT_ID=$(az ad user show \
  --id $USER_EMAIL `# User email or UPN` \
  --query 'id' `# Extract object ID (GUID)` \
  --output tsv)

echo "User Object ID: $USER_OBJECT_ID"

# Validate user exists
if [ -z "$USER_OBJECT_ID" ]; then
  echo "ERROR: User not found. Please verify USER_EMAIL is correct."
  exit 1
fi
```

### Get Cluster Resource ID:

```bash
# Retrieve AKS cluster resource ID dynamically
echo ""
echo "=== Retrieving Cluster Resource ID ==="
CLUSTER_ID=$(az aks show \
  --resource-group $RESOURCE_GROUP `# Source resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --query 'id' `# Extract full Azure resource ID` \
  --output tsv)

echo "Cluster ID: $CLUSTER_ID"

# Validate cluster ID retrieved
if [ -z "$CLUSTER_ID" ]; then
  echo "ERROR: Cluster not found. Please verify cluster was created successfully."
  exit 1
fi
```

### Assign Azure Kubernetes Service Cluster User Role:

```bash
# Assign Azure RBAC role for API server access
echo ""
echo "=== Assigning Azure RBAC Role ==="
ROLE_ASSIGNMENT=$(az role assignment create \
  --assignee $USER_OBJECT_ID `# Target user (Entra ID Object ID)` \
  --role "Azure Kubernetes Service Cluster User Role" `# Built-in role for API authentication` \
  --scope $CLUSTER_ID `# Cluster-level scope (full resource ID)` \
  --query 'roleDefinitionName' `# Extract assigned role name` \
  --output tsv)

echo "Role assigned: $ROLE_ASSIGNMENT"

# Verify role assignment
echo ""
echo "=== Verifying Role Assignment ==="
az role assignment list \
  --assignee $USER_OBJECT_ID \
  --scope $CLUSTER_ID \
  --output table
```

---

## 4. Authenticate Using Entra ID

```bash
# Login with device code (if not already logged in)
echo "=== Authenticating with Azure ==="
az login --use-device-code

# Get AKS credentials (triggers Entra ID authentication)
echo ""
echo "=== Getting AKS Credentials ==="
az aks get-credentials \
  --resource-group $RESOURCE_GROUP `# Source resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --overwrite-existing `# Replace existing kubeconfig entry` \
  --output none

# Verify current context
CURRENT_CONTEXT=$(kubectl config current-context)
echo "Current kubectl context: $CURRENT_CONTEXT"

# Test authentication (should prompt for browser login if not cached)
echo ""
echo "=== Testing Cluster Access ==="
kubectl get nodes 2>&1 || echo "Note: You may see 'Forbidden' - this is expected without Kubernetes RBAC configured"
```

**Expected behavior:**
- First time: Browser opens for Entra ID authentication
- Azure RBAC allows API server access
- Kubernetes RBAC determines actual permissions (currently none)

### Verify Current Identity:

```bash
# Check authenticated Azure account
echo ""
echo "=== Current Azure Identity ==="
az account show \
  --query 'user' \
  --output table

# Check Kubernetes identity (requires kubectl 1.26+)
echo ""
echo "=== Current Kubernetes Identity ==="
kubectl auth whoami 2>/dev/null || echo "kubectl auth whoami requires kubectl 1.26+"
```

---

---

## 5. Configure Kubernetes RBAC (Namespace-Scoped Permissions)

Kubernetes RBAC controls **authorization** within the cluster after Azure RBAC grants API access.

### Create Namespace:

```bash
# Create namespace for RBAC testing
echo "=== Creating Namespace ==="
kubectl create namespace $NAMESPACE `# Create isolated namespace`

# Verify namespace creation
NAMESPACE_CREATED=$(kubectl get namespace $NAMESPACE \
  --output jsonpath='{.metadata.name}')

echo "Namespace created: $NAMESPACE_CREATED"

# Validate namespace exists
if [ "$NAMESPACE_CREATED" != "$NAMESPACE" ]; then
  echo "ERROR: Namespace creation failed."
  exit 1
fi
```

### Create Role with Permissions:

```bash
# Create Kubernetes Role with development permissions
echo ""
echo "=== Creating Kubernetes Role ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: $ROLE_NAME
  namespace: $NAMESPACE
rules:
- apiGroups: ["", "apps", "batch"]  # Core, apps, and batch API groups
  resources: ["pods", "deployments", "jobs", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]  # Full CRUD operations
- apiGroups: [""]
  resources: ["pods/log", "pods/status"]  # Subresources for debugging
  verbs: ["get", "list"]
EOF

# Verify role creation
echo ""
echo "=== Created Role ==="
kubectl get role -n $NAMESPACE

# Describe role details
echo ""
echo "=== Role Details ==="
kubectl describe role $ROLE_NAME -n $NAMESPACE
```

### Create RoleBinding to Entra ID User:

```bash
# Bind role to Entra ID user
echo ""
echo "=== Creating RoleBinding ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: $ROLEBINDING_NAME
  namespace: $NAMESPACE
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $ROLE_NAME
subjects:
- kind: User
  name: $USER_OBJECT_ID  # Entra ID Object ID used for authentication
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify RoleBinding creation
echo ""
echo "=== Created RoleBinding ==="
kubectl get rolebinding -n $NAMESPACE

# Describe RoleBinding details
echo ""
echo "=== RoleBinding Details ==="
kubectl describe rolebinding $ROLEBINDING_NAME -n $NAMESPACE
```

---

## 6. Validate Permissions

### Test Allowed Operations (Should Succeed):

```bash
echo "=== Testing Allowed Operations ==="

# List pods in rbac-lab namespace (should succeed)
echo "1. Listing pods in $NAMESPACE..."
kubectl get pods -n $NAMESPACE

# Create deployment (should succeed)
echo ""
echo "2. Creating deployment..."
DEPLOYMENT_RESULT=$(kubectl create deployment nginx `# Create deployment` \
  --image=nginx `# Use nginx container image` \
  --replicas=2 `# Start with 2 replicas` \
  -n $NAMESPACE) `# Target namespace`
echo "$DEPLOYMENT_RESULT"

# Wait for deployment to be ready
kubectl wait --for=condition=available `# Wait for deployment availability` \
  --timeout=60s `# Timeout after 60 seconds` \
  deployment/nginx `# Target deployment` \
  -n $NAMESPACE `# Target namespace` \
  2>/dev/null || echo "Deployment may take time to become ready"

# List deployments (should succeed)
echo ""
echo "3. Listing deployments..."
kubectl get deployments -n $NAMESPACE

# View deployment details (should succeed)
echo ""
echo "4. Describing deployment..."
kubectl describe deployment nginx -n $NAMESPACE | head -20

# Get pod logs (should succeed)
echo ""
echo "5. Getting pod logs..."
POD_NAME=$(kubectl get pods -n $NAMESPACE \
  -l app=nginx \
  --output jsonpath='{.items[0].metadata.name}' 2>/dev/null)

if [ ! -z "$POD_NAME" ]; then
  kubectl logs $POD_NAME -n $NAMESPACE --tail=5 2>/dev/null || echo "Pod not yet ready"
fi
```

### Test Denied Operations (Should Fail):

```bash
echo ""
echo "=== Testing Denied Operations ==="

# Try to list pods in default namespace (should fail)
echo "1. Attempting to list pods in default namespace..."
kubectl get pods -n default 2>&1 | tee /tmp/rbac-test-default.txt
if grep -q "forbidden" /tmp/rbac-test-default.txt; then
  echo "✓ Correctly denied access to default namespace"
else
  echo "✗ Unexpected: Access was granted to default namespace"
fi

# Try to list nodes (cluster-scoped, should fail)
echo ""
echo "2. Attempting to list nodes..."
kubectl get nodes 2>&1 | tee /tmp/rbac-test-nodes.txt
if grep -q "forbidden" /tmp/rbac-test-nodes.txt; then
  echo "✓ Correctly denied access to cluster-scoped resources"
else
  echo "✗ Unexpected: Access was granted to nodes"
fi

# Try to create namespace (cluster-scoped, should fail)
echo ""
echo "3. Attempting to create namespace..."
kubectl create namespace test-unauthorized 2>&1 | tee /tmp/rbac-test-ns.txt
if grep -q "forbidden" /tmp/rbac-test-ns.txt; then
  echo "✓ Correctly denied cluster-scoped operations"
else
  echo "✗ Unexpected: Namespace creation was allowed"
fi
```

### View Effective Permissions:

```bash
# Check what you can do in rbac-lab namespace
echo ""
echo "=== Effective Permissions in $NAMESPACE ==="
kubectl auth can-i --list -n $NAMESPACE

# Check specific permissions
echo ""
echo "=== Testing Specific Permissions ==="
echo -n "Create pods in $NAMESPACE: "
kubectl auth can-i create pods -n $NAMESPACE

echo -n "Create pods in default: "
kubectl auth can-i create pods -n default

echo -n "Delete nodes: "
kubectl auth can-i delete nodes

echo -n "List deployments in $NAMESPACE: "
kubectl auth can-i list deployments -n $NAMESPACE
```

Expected results:
- **Allowed** in `rbac-lab` namespace
- **Denied** in `default` and other namespaces
- **Denied** for cluster-scoped resources (nodes, namespaces, etc.)

---

## 7. Optional: Group-Based RBAC with Entra ID

### Create Entra ID Group:

```bash
# Create Entra ID group for team-based access management
echo "=== Creating Entra ID Group ==="
GROUP_ID=$(az ad group create \
  --display-name $GROUP_NAME `# Group display name` \
  --mail-nickname $GROUP_NAME `# Email alias (must be unique)` \
  --query 'id' `# Extract group object ID (GUID)` \
  --output tsv)

echo "Group ID: $GROUP_ID"

# Validate group creation
if [ -z "$GROUP_ID" ]; then
  echo "ERROR: Group creation failed."
  exit 1
fi
```

### Add User to Group:

```bash
# Add user to the Entra ID group
echo ""
echo "=== Adding User to Group ==="
az ad group member add \
  --group $GROUP_ID `# Target group (object ID)` \
  --member-id $USER_OBJECT_ID `# User to add (object ID)` \
  --output none

# Verify group membership
echo ""
echo "=== Group Members ==="
az ad group member list \
  --group $GROUP_ID `# Source group` \
  --query '[].{Name:displayName, Email:userPrincipalName, ObjectId:id}' `# Extract member details` \
  --output table
```

### Create Group-Based RoleBinding:

```bash
# Bind role to Entra ID group (enables group-based access control)
echo ""
echo "=== Creating Group-Based RoleBinding ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding-group
  namespace: $NAMESPACE
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $ROLE_NAME  # Reference to the Role created earlier
subjects:
- kind: Group  # Bind to Entra ID group (not individual users)
  name: $GROUP_ID  # Entra ID Group Object ID (GUID)
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify group-based RoleBinding
echo ""
echo "=== Group-Based RoleBinding ==="
kubectl get rolebinding dev-rolebinding-group -n $NAMESPACE

# Describe RoleBinding
echo ""
kubectl describe rolebinding dev-rolebinding-group -n $NAMESPACE
```

**Note:** Group membership changes may take 5-10 minutes to propagate to Kubernetes tokens. Users may need to re-authenticate (`az aks get-credentials` again) to pick up new group memberships.

---

## 8. View All RBAC Configurations

```bash
# View Azure role assignments for the cluster
echo "=== Azure RBAC Assignments ==="
az role assignment list \
  --scope $CLUSTER_ID `# Filter by cluster scope` \
  --query "[?principalType=='User'].{User:principalName, Role:roleDefinitionName, Scope:scope}" `# Format output` \
  --output table

# View Kubernetes RoleBindings
echo ""
echo "=== Kubernetes RoleBindings in $NAMESPACE ==="
kubectl get rolebindings -n $NAMESPACE

# View Kubernetes Roles
echo ""
echo "=== Kubernetes Roles in $NAMESPACE ==="
kubectl get roles -n $NAMESPACE

# View ClusterRoleBindings (if any)
echo ""
echo "=== Azure-Related ClusterRoleBindings ==="
kubectl get clusterrolebindings | grep -i azure || echo "No Azure-related ClusterRoleBindings found"
```

---

## Expected Results

✅ AKS cluster created with Entra ID integration enabled  
✅ Azure RBAC role assigned for API server access  
✅ Entra ID authentication working (browser-based or device code)  
✅ Kubernetes RBAC configured at namespace level  
✅ User can perform operations in `rbac-lab` namespace  
✅ User denied access to other namespaces and cluster-scoped resources  
✅ `kubectl auth can-i` correctly reflects permission boundaries  
✅ Group-based RBAC (optional) functioning correctly  

---

## How This Connects to Enterprise AKS Governance

This lab demonstrates **defense-in-depth authentication and authorization**:

### Two-Layer Security Model:

```
User Authentication Request
    ↓
Layer 1: Azure RBAC (API Server Access)
    ├─ Validates Entra ID identity
    ├─ Checks "Cluster User Role"
    └─ Grants/denies API server access
    ↓
Layer 2: Kubernetes RBAC (Resource Authorization)
    ├─ Verifies Role/RoleBinding
    ├─ Checks namespace scope
    └─ Grants/denies specific operations
    ↓
Resource Access Granted/Denied
```

### Enterprise Use Cases:

| Persona | Azure RBAC | Kubernetes RBAC | Access Pattern |
|---------|-----------|-----------------|----------------|
| **Platform Admin** | Owner | ClusterRoleBinding (cluster-admin) | Full cluster access |
| **Dev Team Lead** | Cluster User | RoleBinding (admin) in team namespace | Full namespace access |
| **Developer** | Cluster User | RoleBinding (edit) in team namespace | Read/write pods, services |
| **Auditor** | Cluster User | ClusterRoleBinding (view) | Read-only all namespaces |

### Integration with Session 2 Labs:

- **Lab 2a (Azure Policy):** Policies enforced **before** RBAC checks
- **Lab 2b (KEDA):** RBAC controls who can create ScaledObjects
- **Lab 2c (This Lab):** RBAC controls who can access namespaces
- **Combined:** Complete governance framework (Policy → RBAC → Admission Control)

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete namespace (removes all resources within including deployments, pods, roles, rolebindings)
kubectl delete namespace $NAMESPACE `# Cascading delete of all namespace resources`

echo "Namespace and all resources deleted"
```

### Option 2: Remove RBAC Assignments
```bash
# Remove Azure RBAC role assignment
az role assignment delete \
  --assignee $USER_OBJECT_ID `# User object ID` \
  --scope $CLUSTER_ID `# Cluster scope` \
  --output none

echo "Azure RBAC role assignment removed"

# Delete Entra ID group (if created)
if [ ! -z "$GROUP_ID" ]; then
  az ad group delete \
    --group $GROUP_ID `# Group object ID` \
    --output none 2>/dev/null && echo "Entra ID group deleted" || echo "No group to delete"
fi
```

### Option 3: Delete Entire Resource Group (Complete Cleanup)
```bash
# Delete entire resource group (removes all resources: cluster, nodes, networking, etc.)
echo "Deleting resource group: $RESOURCE_GROUP..."
DELETE_RESULT=$(az group delete \
  --name $RESOURCE_GROUP `# Target resource group` \
  --yes `# Skip confirmation prompt` \
  --no-wait `# Run asynchronously (don't wait for completion)` \
  --output tsv)

echo "Delete operation initiated: $DELETE_RESULT"

# Verify deletion status (run after a few minutes)
echo ""
echo "To verify deletion status, run:"
echo "az group show --name $RESOURCE_GROUP --query properties.provisioningState"
```

---

## Key Takeaways

- **Entra ID integration** provides centralized identity and authentication for AKS
- **Azure RBAC** controls authentication and initial authorization to the Kubernetes API server
- **Kubernetes RBAC** controls fine-grained authorization within the cluster
- **Both layers must be configured** for actual resource access
- **Namespace-scoped RBAC** (Role/RoleBinding) is the foundation of multi-team AKS governance
- **Cluster-scoped RBAC** (ClusterRole/ClusterRoleBinding) for platform administrators
- **Group-based access** simplifies management across multiple users
- Authentication tokens are cached and auto-refreshed by Azure CLI and kubeconfig
- **Conditional Access policies** (MFA, device compliance) are enforced transparently
- `kubectl auth can-i` is essential for testing and validating permissions

---

## Azure RBAC Built-in Roles for AKS

| Role Name | Scope | Permissions |
|-----------|-------|-------------|
| **Azure Kubernetes Service Cluster Admin Role** | Cluster | Full cluster access including admin credentials |
| **Azure Kubernetes Service Cluster User Role** | Cluster | Can get user credentials, requires K8s RBAC for resources |
| **Azure Kubernetes Service RBAC Admin** | Namespace/Cluster | Full Kubernetes RBAC permissions |
| **Azure Kubernetes Service RBAC Writer** | Namespace/Cluster | Read/write Kubernetes resources (pods, services, deployments) |
| **Azure Kubernetes Service RBAC Reader** | Namespace/Cluster | Read-only access to Kubernetes resources |

---

## Troubleshooting

- **Authentication fails** → Check Entra ID integration: `az aks show --resource-group $RG --name $AKS_NAME --query aadProfile`
- **Permissions denied after Azure RBAC assignment** → Verify Kubernetes RBAC Role/RoleBinding exists
- **Group membership not working** → Allow 5-10 minutes for token refresh; re-run `az aks get-credentials`
- **Can't get credentials** → Ensure "Azure Kubernetes Service Cluster User Role" is assigned
- **Browser not opening** → Use `az login --use-device-code` for device code flow
- **Forbidden on all operations** → Check both Azure RBAC AND Kubernetes RBAC configurations
- **kubectl auth whoami not working** → Requires kubectl 1.26+; use `az account show` instead

---

## Additional Resources

- [AKS-managed Entra ID integration](https://learn.microsoft.com/en-us/azure/aks/managed-aad)
- [Azure RBAC for Kubernetes Authorization](https://learn.microsoft.com/en-us/azure/aks/manage-azure-rbac)
- [Kubernetes RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Best practices for authentication and authorization in AKS](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-identity)

---


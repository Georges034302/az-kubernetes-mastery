# Lab 2d: Namespace-Scoped RBAC for Multi-Team Isolation

## Objective
Create a complete multi-tenant AKS environment with **namespace-scoped RBAC** for multiple teams, demonstrating proper isolation, resource quotas, and least-privilege access patterns for production-grade Kubernetes governance.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Azure configuration
RESOURCE_GROUP="rg-aks-namespace-rbac-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-namespace-rbac-demo"

# Cluster configuration
NODE_COUNT=3
NODE_VM_SIZE="Standard_DS2_v2"

# Namespace configuration
NS_FRONTEND="team-frontend"
NS_BACKEND="team-backend"
NS_DATA="team-data"
NS_SHARED="shared-services"

# Service account names
SA_FRONTEND="frontend-team-sa"
SA_BACKEND="backend-team-sa"
SA_DATA="data-team-sa"

# Role names
ROLE_FRONTEND="frontend-developer"
ROLE_BACKEND="backend-developer"
ROLE_DATA="data-engineer"
ROLE_SHARED_READER="shared-reader"

# Display configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Namespaces: $NS_FRONTEND, $NS_BACKEND, $NS_DATA, $NS_SHARED"
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

echo "Resource Group creation: $RG_RESULT"

# Create AKS cluster
echo ""
echo "Creating AKS cluster (this may take 5-10 minutes)..."
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_VM_SIZE `# Node VM size` \
  --enable-managed-identity `# Use managed identity` \
  --generate-ssh-keys `# Auto-generate SSH keys` \
  --query 'provisioningState' \
  --output tsv)

echo "AKS cluster creation: $CLUSTER_RESULT"

# Get AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP `# Source resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --overwrite-existing `# Replace existing kubeconfig entry` \
  --output none

# Verify cluster connectivity
CURRENT_CONTEXT=$(kubectl config current-context)
echo "Current kubectl context: $CURRENT_CONTEXT"

# Display cluster nodes
echo ""
echo "Cluster nodes:"
kubectl get nodes
```

---

## 3. Create Namespaces for Different Teams

```bash
# Create team namespaces
echo "=== Creating Team Namespaces ==="

kubectl create namespace $NS_FRONTEND `# Frontend team namespace`
kubectl create namespace $NS_BACKEND `# Backend team namespace`
kubectl create namespace $NS_DATA `# Data engineering team namespace`
kubectl create namespace $NS_SHARED `# Shared services namespace`

# Add descriptive labels for organization and filtering
echo ""
echo "=== Labeling Namespaces ==="

kubectl label namespace $NS_FRONTEND \
  team=frontend \
  env=production \
  tier=web \
  --overwrite

kubectl label namespace $NS_BACKEND \
  team=backend \
  env=production \
  tier=api \
  --overwrite

kubectl label namespace $NS_DATA \
  team=data \
  env=production \
  tier=analytics \
  --overwrite

kubectl label namespace $NS_SHARED \
  tier=shared \
  env=production \
  purpose=common-services \
  --overwrite

# Verify namespace creation and labels
echo ""
echo "=== Created Namespaces with Labels ==="
kubectl get namespaces \
  -l env=production \
  --show-labels
```

---

## 4. Create Service Accounts for Each Team

```bash
# Create service accounts for team-specific workloads
echo ""
echo "=== Creating Service Accounts ==="

kubectl create serviceaccount $SA_FRONTEND \
  -n $NS_FRONTEND `# Frontend team service account`

kubectl create serviceaccount $SA_BACKEND \
  -n $NS_BACKEND `# Backend team service account`

kubectl create serviceaccount $SA_DATA \
  -n $NS_DATA `# Data team service account`

# Verify service account creation
echo ""
echo "=== Created Service Accounts ==="
echo "Frontend SA:"
kubectl get serviceaccount $SA_FRONTEND -n $NS_FRONTEND

echo ""
echo "Backend SA:"
kubectl get serviceaccount $SA_BACKEND -n $NS_BACKEND

echo ""
echo "Data SA:"
kubectl get serviceaccount $SA_DATA -n $NS_DATA
```

---

## 5. Create Namespace-Scoped Roles

### Frontend Team Role:

```bash
# Create role for frontend developers
echo ""
echo "=== Creating Frontend Developer Role ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: $ROLE_FRONTEND
  namespace: $NS_FRONTEND
rules:
# Pods - full access for debugging
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "delete", "exec"]
# Deployments - full lifecycle management
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Services - networking configuration
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# ConfigMaps - full access for application configuration
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Secrets - read-only for security
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
# Ingress - manage external routing
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

echo "Frontend developer role created"
```

### Backend Team Role:

```bash
# Create role for backend developers
echo ""
echo "=== Creating Backend Developer Role ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: $ROLE_BACKEND
  namespace: $NS_BACKEND
rules:
# Pods - access for application management
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch", "create", "delete"]
# Deployments and StatefulSets - full control for backend services
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Services and Endpoints - networking management
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# ConfigMaps and Secrets - full access for backend configuration
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Batch workloads - jobs and cron jobs
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

echo "Backend developer role created"
```

### Data Team Role:

```bash
# Create role for data engineers
echo ""
echo "=== Creating Data Engineer Role ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: $ROLE_DATA
  namespace: $NS_DATA
rules:
# Pods - for data processing workloads
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch", "create", "delete"]
# StatefulSets - for databases and stateful data services
- apiGroups: ["apps"]
  resources: ["statefulsets", "deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Batch jobs - ETL and data pipelines
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# PersistentVolumeClaims - data storage management
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# ConfigMaps and Secrets - configuration for data services
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Services - database endpoints
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

echo "Data engineer role created"
```

---

## 6. Create RoleBindings

### Frontend Team RoleBinding:

```bash
# Bind frontend role to service account
echo "=== Creating Frontend Team RoleBinding ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-binding
  namespace: $NS_FRONTEND
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $ROLE_FRONTEND
subjects:
- kind: ServiceAccount
  name: $SA_FRONTEND
  namespace: $NS_FRONTEND
EOF

echo "Frontend team RoleBinding created"
```

### Backend Team RoleBinding:

```bash
# Bind backend role to service account
echo ""
echo "=== Creating Backend Team RoleBinding ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-team-binding
  namespace: $NS_BACKEND
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $ROLE_BACKEND
subjects:
- kind: ServiceAccount
  name: $SA_BACKEND
  namespace: $NS_BACKEND
EOF

echo "Backend team RoleBinding created"
```

### Data Team RoleBinding:

```bash
# Bind data role to service account
echo ""
echo "=== Creating Data Team RoleBinding ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: data-team-binding
  namespace: $NS_DATA
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $ROLE_DATA
subjects:
- kind: ServiceAccount
  name: $SA_DATA
  namespace: $NS_DATA
EOF

echo "Data team RoleBinding created"
```

### Verify All Roles and RoleBindings:

```bash
# Verify all created roles
echo ""
echo "=== All Created Roles ==="
kubectl get roles -A | grep -E "frontend-developer|backend-developer|data-engineer"

# Verify all created rolebindings
echo ""
echo "=== All Created RoleBindings ==="
kubectl get rolebindings -A | grep -E "frontend-team|backend-team|data-team"
```

---

## 7. Create Read-Only Role for Shared Services

```bash
# Create read-only role for shared services
echo ""
echo "=== Creating Shared Services Reader Role ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: $ROLE_SHARED_READER
  namespace: $NS_SHARED
rules:
# Read-only access to services and endpoints
- apiGroups: [""]
  resources: ["services", "endpoints", "configmaps"]
  verbs: ["get", "list", "watch"]
# Read-only access to deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EOF

echo "Shared services reader role created"

# Create RoleBinding allowing all teams to read shared services
echo ""
echo "=== Creating Shared Services RoleBinding ==="
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: all-teams-shared-access
  namespace: $NS_SHARED
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $ROLE_SHARED_READER
subjects:
# Frontend team access
- kind: ServiceAccount
  name: $SA_FRONTEND
  namespace: $NS_FRONTEND
# Backend team access
- kind: ServiceAccount
  name: $SA_BACKEND
  namespace: $NS_BACKEND
# Data team access
- kind: ServiceAccount
  name: $SA_DATA
  namespace: $NS_DATA
EOF

echo "All teams granted read access to shared services"
```

---

## 8. Deploy Test Resources

### Deploy sample applications in each namespace:

```bash
# Deploy frontend application
echo ""
echo "=== Deploying Frontend Application ==="
kubectl create deployment nginx-frontend \
  --image=nginx \
  --replicas=2 \
  -n $NS_FRONTEND

kubectl expose deployment nginx-frontend \
  --port=80 \
  --target-port=80 \
  -n $NS_FRONTEND

# Deploy backend application
echo ""
echo "=== Deploying Backend Application ==="
kubectl create deployment api-backend \
  --image=nginx \
  --replicas=2 \
  -n $NS_BACKEND

kubectl expose deployment api-backend \
  --port=8080 \
  --target-port=80 \
  -n $NS_BACKEND

# Deploy data processing job
echo ""
echo "=== Deploying Data Processing Job ==="
kubectl create job data-processor \
  --image=busybox \
  -n $NS_DATA \
  -- sh -c "echo 'Processing data...' && sleep 30"

# Deploy shared service
echo ""
echo "=== Deploying Shared Service ==="
kubectl create deployment monitoring \
  --image=prom/prometheus \
  --replicas=1 \
  -n $NS_SHARED

kubectl expose deployment monitoring \
  --port=9090 \
  --target-port=9090 \
  -n $NS_SHARED

# Wait for deployments to be ready
echo ""
echo "Waiting for deployments to be ready..."
kubectl wait --for=condition=available \
  --timeout=60s \
  deployment/nginx-frontend \
  -n $NS_FRONTEND 2>/dev/null || echo "Frontend deployment starting..."

kubectl wait --for=condition=available \
  --timeout=60s \
  deployment/api-backend \
  -n $NS_BACKEND 2>/dev/null || echo "Backend deployment starting..."
```

---

## 9. Test Service Account Permissions

### Create test pod using frontend service account:

```bash
# Create test pod with frontend service account
echo ""
echo "=== Creating Test Pod with Frontend Service Account ==="
kubectl run test-pod-frontend \
  --image=bitnami/kubectl `# Container with kubectl pre-installed` \
  --serviceaccount=$SA_FRONTEND `# Use frontend service account` \
  -n $NS_FRONTEND \
  -- sleep 3600 `# Keep pod running for testing`

# Wait for pod to be ready
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/test-pod-frontend \
  -n $NS_FRONTEND

echo "Test pod ready"
```

### Test allowed operations (should succeed):

```bash
# Test operations that SHOULD work
echo ""
echo "=== Testing Allowed Operations ==="

# List pods in own namespace (should succeed)
echo "1. Listing pods in frontend namespace:"
kubectl exec test-pod-frontend -n $NS_FRONTEND -- \
  kubectl get pods -n $NS_FRONTEND

# Create deployment in own namespace (should succeed)
echo ""
echo "2. Creating deployment in frontend namespace:"
kubectl exec test-pod-frontend -n $NS_FRONTEND -- \
  kubectl create deployment test-deploy --image=nginx -n $NS_FRONTEND

# View shared services (should succeed - read-only)
echo ""
echo "3. Viewing shared services:"
kubectl exec test-pod-frontend -n $NS_FRONTEND -- \
  kubectl get services -n $NS_SHARED
```

### Test denied operations (should fail):

```bash
# Test operations that SHOULD fail
echo ""
echo "=== Testing Denied Operations ==="

# Try to list pods in backend namespace (should fail)
echo "1. Attempting to list pods in backend namespace:"
kubectl exec test-pod-frontend -n $NS_FRONTEND -- \
  kubectl get pods -n $NS_BACKEND 2>&1 | grep -i "forbidden" && \
  echo "✓ Correctly denied access to backend namespace" || \
  echo "✗ Unexpected: Access was granted"

# Try to create resources in shared namespace (should fail - read-only)
echo ""
echo "2. Attempting to create deployment in shared namespace:"
kubectl exec test-pod-frontend -n $NS_FRONTEND -- \
  kubectl create deployment unauthorized --image=nginx -n $NS_SHARED 2>&1 | \
  grep -i "forbidden" && \
  echo "✓ Correctly denied write access to shared namespace" || \
  echo "✗ Unexpected: Write access was granted"

# Try to list nodes (cluster-scoped, should fail)
echo ""
echo "3. Attempting to list nodes:"
kubectl exec test-pod-frontend -n $NS_FRONTEND -- \
  kubectl get nodes 2>&1 | grep -i "forbidden" && \
  echo "✓ Correctly denied cluster-scoped access" || \
  echo "✗ Unexpected: Cluster access was granted"
```

---

## 10. Test Permissions Using kubectl auth can-i

```bash
# Test permissions for frontend service account
echo ""
echo "=== Testing Frontend Service Account Permissions ==="

# Test specific permissions
echo "Can frontend SA create pods in frontend namespace?"
kubectl auth can-i create pods \
  --as=system:serviceaccount:$NS_FRONTEND:$SA_FRONTEND \
  -n $NS_FRONTEND

echo ""
echo "Can frontend SA create pods in backend namespace?"
kubectl auth can-i create pods \
  --as=system:serviceaccount:$NS_FRONTEND:$SA_FRONTEND \
  -n $NS_BACKEND

echo ""
echo "Can frontend SA list services in shared namespace?"
kubectl auth can-i list services \
  --as=system:serviceaccount:$NS_FRONTEND:$SA_FRONTEND \
  -n $NS_SHARED

echo ""
echo "Can frontend SA create deployments in shared namespace?"
kubectl auth can-i create deployments \
  --as=system:serviceaccount:$NS_FRONTEND:$SA_FRONTEND \
  -n $NS_SHARED

# List all permissions for frontend SA in its namespace
echo ""
echo "=== All Permissions for Frontend SA in $NS_FRONTEND ==="
kubectl auth can-i --list \
  --as=system:serviceaccount:$NS_FRONTEND:$SA_FRONTEND \
  -n $NS_FRONTEND
```

---

## 11. Apply Resource Quotas (Optional)

```bash
# Apply resource quotas to prevent resource exhaustion
echo ""
echo "=== Applying Resource Quotas ==="

# Frontend namespace quota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: frontend-quota
  namespace: $NS_FRONTEND
spec:
  hard:
    requests.cpu: "4"  # Maximum 4 CPU cores requested
    requests.memory: "8Gi"  # Maximum 8GB memory requested
    limits.cpu: "8"  # Maximum 8 CPU cores limit
    limits.memory: "16Gi"  # Maximum 16GB memory limit
    pods: "20"  # Maximum 20 pods
    services: "10"  # Maximum 10 services
    persistentvolumeclaims: "5"  # Maximum 5 PVCs
EOF

# Backend namespace quota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-quota
  namespace: $NS_BACKEND
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    pods: "30"
    services: "15"
    persistentvolumeclaims: "10"
EOF

# Data namespace quota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: data-quota
  namespace: $NS_DATA
spec:
  hard:
    requests.cpu: "16"
    requests.memory: "64Gi"
    limits.cpu: "32"
    limits.memory: "128Gi"
    pods: "50"
    services: "5"
    persistentvolumeclaims: "20"
EOF

# Verify quotas
echo ""
echo "=== Resource Quotas Applied ==="
kubectl get resourcequota -A

# View detailed quota usage
echo ""
echo "=== Frontend Namespace Quota Details ==="
kubectl describe resourcequota frontend-quota -n $NS_FRONTEND
```

---

## 12. Audit RBAC Configuration

```bash
# Audit all RBAC configurations
echo ""
echo "=== RBAC Audit Summary ==="

# List all roles across namespaces
echo "All Roles:"
kubectl get roles -A

# List all rolebindings
echo ""
echo "All RoleBindings:"
kubectl get rolebindings -A

# List all service accounts
echo ""
echo "All Service Accounts:"
kubectl get serviceaccounts -A | grep -E "frontend|backend|data"

# View detailed role permissions
echo ""
echo "=== Detailed Frontend Role Permissions ==="
kubectl describe role $ROLE_FRONTEND -n $NS_FRONTEND

echo ""
echo "=== Detailed Backend Role Permissions ==="
kubectl describe role $ROLE_BACKEND -n $NS_BACKEND

echo ""
echo "=== Detailed Data Role Permissions ==="
kubectl describe role $ROLE_DATA -n $NS_DATA
```

---

## Expected Results

✅ Four namespaces created with proper labels  
✅ Service accounts created for each team  
✅ Namespace-scoped Roles with team-specific permissions  
✅ RoleBindings linking service accounts to roles  
✅ Shared services accessible (read-only) to all teams  
✅ Teams can manage resources in their own namespaces  
✅ Teams cannot access other teams' namespaces  
✅ Resource quotas enforced per namespace  
✅ `kubectl auth can-i` correctly reflects permission boundaries  

---

## How This Connects to Enterprise Multi-Tenancy

This lab demonstrates **production-grade namespace isolation** for multi-team Kubernetes environments:

### Namespace Isolation Strategy:

```
Team Namespaces (Isolated)
├─ team-frontend (Web Applications)
├─ team-backend (API Services)
└─ team-data (Data Processing)

Shared Namespace (Read-Only Cross-Access)
└─ shared-services (Monitoring, Logging, etc.)

RBAC Enforcement Layers:
1. Service Accounts → Workload Identity
2. Roles → Permission Definitions
3. RoleBindings → Grant Access
4. Resource Quotas → Prevent Resource Starvation
```

### Enterprise Use Cases:

| Pattern | Implementation | Benefit |
|---------|---------------|---------|
| **Team Isolation** | Namespace-scoped Roles | Prevents accidental cross-team interference |
| **Shared Services** | Read-only RoleBindings across namespaces | Central monitoring/logging for all teams |
| **Resource Governance** | ResourceQuotas per namespace | Fair resource allocation, prevent noisy neighbors |
| **Least Privilege** | Minimal verbs per role | Reduces blast radius of compromised accounts |
| **Service Identity** | ServiceAccounts for pods | Workload-level authentication and authorization |

---

## Cleanup

### Option 1: Delete Deployments and Jobs Only
```bash
# Delete test resources
echo "=== Deleting Test Resources ==="

kubectl delete deployment nginx-frontend -n $NS_FRONTEND
kubectl delete deployment test-deploy -n $NS_FRONTEND --ignore-not-found
kubectl delete deployment api-backend -n $NS_BACKEND
kubectl delete job data-processor -n $NS_DATA --ignore-not-found
kubectl delete deployment monitoring -n $NS_SHARED
kubectl delete pod test-pod-frontend -n $NS_FRONTEND --ignore-not-found

echo "Test resources deleted"
```

### Option 2: Delete Namespaces (Cascading Delete)
```bash
# Delete all team namespaces (removes all resources within)
echo "=== Deleting All Team Namespaces ==="

kubectl delete namespace $NS_FRONTEND `# Deletes all frontend resources`
kubectl delete namespace $NS_BACKEND `# Deletes all backend resources`
kubectl delete namespace $NS_DATA `# Deletes all data resources`
kubectl delete namespace $NS_SHARED `# Deletes all shared resources`

echo "All namespaces and resources deleted"
```

### Option 3: Delete Entire Resource Group (Complete Cleanup)
```bash
# Delete entire resource group (removes cluster and all resources)
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

- **Namespace isolation** is the foundation of multi-team AKS governance
- **Service Accounts** provide workload-level identity for pod authentication
- **Roles** define permissions at the namespace level (not cluster-wide)
- **RoleBindings** connect identities (users, groups, service accounts) to Roles
- **Shared namespaces** enable common services accessible to all teams
- **Resource Quotas** prevent resource exhaustion and ensure fair allocation
- **Least privilege** principle: grant minimum required permissions
- `kubectl auth can-i` is essential for testing and validating RBAC
- **Labels** on namespaces enable organized filtering and policy enforcement
- Namespace-scoped RBAC scales better than cluster-scoped for multi-tenancy

---

## RBAC Best Practices

| Practice | Implementation | Rationale |
|----------|---------------|-----------|
| **Use ServiceAccounts for pods** | Bind roles to SAs, not default | Principle of least privilege |
| **Separate namespaces by team/env** | team-frontend, team-backend | Clear isolation boundaries |
| **Read-only shared services** | Separate role with limited verbs | Prevent accidental modifications |
| **Limit secret access** | Read-only or no access | Reduce credential exposure |
| **Apply resource quotas** | ResourceQuota per namespace | Prevent resource monopolization |
| **Regular RBAC audits** | kubectl get roles/rolebindings -A | Detect permission creep |
| **Group-based access** | Bind roles to Entra ID groups | Simplify user management |

---

## Troubleshooting

- **Permissions denied** → Check both Role and RoleBinding exist in the correct namespace
- **Service account can't access resources** → Verify RoleBinding subjects match SA name and namespace
- **Cross-namespace access fails** → Remember: Roles are namespace-scoped; use separate RoleBindings
- **Pod can't use service account** → Check serviceAccountName in pod spec matches existing SA
- **Resource quota exceeded** → View quota usage: `kubectl describe resourcequota -n <namespace>`
- **kubectl auth can-i returns unexpected result** → Verify --as flag uses correct format: `system:serviceaccount:<namespace>:<sa-name>`

---

## Additional Resources

- [Kubernetes RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Configure Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Namespace Best Practices](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

---


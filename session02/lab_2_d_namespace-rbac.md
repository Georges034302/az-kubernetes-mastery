# Lab 2d: Namespace-Scoped RBAC

## Objective
Create namespace-scoped RBAC for multi-team isolation.

## Prerequisites
- AKS cluster running
- `kubectl` configured with admin access
- Understanding of Kubernetes RBAC concepts

## Steps

### 1. Create Namespaces for Different Teams
```bash
kubectl create namespace team-frontend
kubectl create namespace team-backend
kubectl create namespace team-data
kubectl create namespace shared-services
```

Add labels for organization:
```bash
kubectl label namespace team-frontend team=frontend env=production
kubectl label namespace team-backend team=backend env=production
kubectl label namespace team-data team=data env=production
kubectl label namespace shared-services tier=shared env=production
```

Verify namespaces:
```bash
kubectl get namespaces --show-labels
```

### 2. Create Service Accounts for Each Team
```bash
kubectl create serviceaccount frontend-team-sa -n team-frontend
kubectl create serviceaccount backend-team-sa -n team-backend
kubectl create serviceaccount data-team-sa -n team-data
```

### 3. Create Namespace-Scoped Roles

**Frontend Team Role** - `frontend-role.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: frontend-developer
  namespace: team-frontend
rules:
# Pods
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "delete", "exec"]
# Deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# ConfigMaps and Secrets (limited)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # Read-only for secrets
# Ingress
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

**Backend Team Role** - `backend-role.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-developer
  namespace: team-backend
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

**Data Team Role** - `data-role.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: data-engineer
  namespace: team-data
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Apply all roles:
```bash
kubectl apply -f frontend-role.yaml
kubectl apply -f backend-role.yaml
kubectl apply -f data-role.yaml
```

### 4. Create RoleBindings

**Frontend RoleBinding** - `frontend-rolebinding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-binding
  namespace: team-frontend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: frontend-developer
subjects:
- kind: ServiceAccount
  name: frontend-team-sa
  namespace: team-frontend
# Add human users
- kind: User
  name: alice@company.com  # Replace with actual Entra ID user
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: bob@company.com
  apiGroup: rbac.authorization.k8s.io
```

**Backend RoleBinding** - `backend-rolebinding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-team-binding
  namespace: team-backend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: backend-developer
subjects:
- kind: ServiceAccount
  name: backend-team-sa
  namespace: team-backend
- kind: User
  name: charlie@company.com
  apiGroup: rbac.authorization.k8s.io
```

**Data RoleBinding** - `data-rolebinding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: data-team-binding
  namespace: team-data
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: data-engineer
subjects:
- kind: ServiceAccount
  name: data-team-sa
  namespace: team-data
- kind: User
  name: dana@company.com
  apiGroup: rbac.authorization.k8s.io
```

Apply RoleBindings:
```bash
kubectl apply -f frontend-rolebinding.yaml
kubectl apply -f backend-rolebinding.yaml
kubectl apply -f data-rolebinding.yaml
```

### 5. Create Read-Only Role for Shared Services
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shared-services-reader
  namespace: shared-services
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

Create RoleBindings for all teams to access shared services:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: all-teams-shared-access
  namespace: shared-services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: shared-services-reader
subjects:
- kind: ServiceAccount
  name: frontend-team-sa
  namespace: team-frontend
- kind: ServiceAccount
  name: backend-team-sa
  namespace: team-backend
- kind: ServiceAccount
  name: data-team-sa
  namespace: team-data
```

```bash
kubectl apply -f shared-services-role.yaml
kubectl apply -f shared-services-rolebinding.yaml
```

### 6. Test Service Account Permissions
Create a test pod using the frontend service account:

```bash
kubectl run test-pod \
  --image=bitnami/kubectl \
  --serviceaccount=frontend-team-sa \
  -n team-frontend \
  -- sleep 3600
```

Test permissions from within the pod:
```bash
# Should work (own namespace)
kubectl exec -it test-pod -n team-frontend -- kubectl get pods -n team-frontend

# Should fail (different namespace)
kubectl exec -it test-pod -n team-frontend -- kubectl get pods -n team-backend

# Should work (shared services - read only)
kubectl exec -it test-pod -n team-frontend -- kubectl get services -n shared-services

# Should fail (shared services - write)
kubectl exec -it test-pod -n team-frontend -- kubectl create deployment nginx --image=nginx -n shared-services
```

### 7. Test with kubectl auth can-i
Impersonate service accounts to test permissions:

```bash
# Test frontend permissions
kubectl auth can-i list pods -n team-frontend --as=system:serviceaccount:team-frontend:frontend-team-sa
kubectl auth can-i delete pods -n team-frontend --as=system:serviceaccount:team-frontend:frontend-team-sa
kubectl auth can-i list pods -n team-backend --as=system:serviceaccount:team-frontend:frontend-team-sa

# Test backend permissions
kubectl auth can-i create jobs -n team-backend --as=system:serviceaccount:team-backend:backend-team-sa
kubectl auth can-i create statefulsets -n team-backend --as=system:serviceaccount:team-backend:backend-team-sa

# List all permissions for a service account
kubectl auth can-i --list -n team-frontend --as=system:serviceaccount:team-frontend:frontend-team-sa
```

### 8. Create Admin Role for Team Leads
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-lead
  namespace: team-frontend
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-lead-binding
  namespace: team-frontend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: team-lead
subjects:
- kind: User
  name: lead@company.com
  apiGroup: rbac.authorization.k8s.io
```

### 9. Implement Resource Quotas per Namespace
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-frontend-quota
  namespace: team-frontend
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    persistentvolumeclaims: "5"
    pods: "50"
    services.loadbalancers: "2"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-frontend-limits
  namespace: team-frontend
spec:
  limits:
  - max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    type: Container
```

Apply quotas to all team namespaces:
```bash
kubectl apply -f frontend-quota.yaml
kubectl apply -f backend-quota.yaml
kubectl apply -f data-quota.yaml
```

View quotas:
```bash
kubectl get resourcequota -n team-frontend
kubectl describe resourcequota team-frontend-quota -n team-frontend
```

### 10. Create Network Policies for Namespace Isolation
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: team-frontend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Allow from same namespace
  - from:
    - namespaceSelector:
        matchLabels:
          tier: shared  # Allow from shared services
```

Apply to each team namespace:
```bash
kubectl apply -f network-policy-frontend.yaml
kubectl apply -f network-policy-backend.yaml
kubectl apply -f network-policy-data.yaml
```

### 11. View and Audit RBAC Configuration
```bash
# List all roles in namespace
kubectl get roles -n team-frontend

# List all rolebindings
kubectl get rolebindings -n team-frontend

# Describe specific role
kubectl describe role frontend-developer -n team-frontend

# Describe rolebinding
kubectl describe rolebinding frontend-team-binding -n team-frontend

# Get all RBAC resources across namespaces
kubectl get roles,rolebindings --all-namespaces | grep team
```

### 12. Test Cross-Namespace Access Restrictions
Deploy test apps in each namespace:

```bash
kubectl run nginx-frontend --image=nginx -n team-frontend
kubectl run nginx-backend --image=nginx -n team-backend

# Test network connectivity
kubectl exec -it nginx-frontend -n team-frontend -- curl nginx-backend.team-backend.svc.cluster.local
```

## Expected Results
- Each team has isolated namespace with specific RBAC permissions
- Service accounts can only access resources in their namespace
- Shared services namespace accessible read-only to all teams
- Resource quotas limit resource consumption per namespace
- Network policies enforce namespace isolation
- `kubectl auth can-i` correctly reflects permissions
- Team leads have full admin access within their namespace

## Cleanup
```bash
kubectl delete namespace team-frontend team-backend team-data shared-services
```

## Key Takeaways
- **Namespaces** provide logical isolation for teams
- **Roles** define permissions within a namespace
- **RoleBindings** assign roles to users or service accounts
- **ResourceQuotas** prevent resource exhaustion
- **LimitRanges** set default and max resource limits
- **NetworkPolicies** add network-level isolation
- **Service accounts** enable pod-level RBAC
- Principle of least privilege: grant minimum necessary permissions

## RBAC Best Practices

| Practice | Description |
|----------|-------------|
| Least Privilege | Grant minimum permissions needed |
| Namespace Scoping | Use Roles instead of ClusterRoles when possible |
| Service Accounts | Create dedicated SAs for applications |
| Group Management | Use Entra ID groups for user management |
| Audit Regularly | Review RBAC with `kubectl auth can-i` |
| Resource Quotas | Prevent resource abuse per namespace |
| Documentation | Document who has access to what |

## Troubleshooting
- **Permission denied**: Check both Role and RoleBinding
- **Service account can't access**: Verify RoleBinding subjects
- **Cross-namespace access**: Expected behavior with namespace-scoped RBAC
- **Quota exceeded**: Check ResourceQuota limits

---

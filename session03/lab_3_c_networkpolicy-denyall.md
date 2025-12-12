# Lab 3c: Network Policy - Deny All (Default-Deny Security Model)

## Objective
Implement **default-deny network policies** in AKS using the zero-trust security model. Demonstrate pod-to-pod isolation, namespace segmentation, and selective traffic allowlisting based on application requirements.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Azure resource configuration
RESOURCE_GROUP="rg-aks-netpol-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-netpol-demo"

# AKS cluster configuration
NODE_COUNT=3
NODE_VM_SIZE="Standard_DS2_v2"
K8S_VERSION="1.28"

# Network policy configuration
NETWORK_PLUGIN="azure"      # Azure CNI required for network policies
NETWORK_POLICY="azure"      # Use Azure Network Policy (alternative: calico)

# Namespace configuration
NS_FRONTEND="frontend"
NS_BACKEND="backend"
NS_DATABASE="database"

# Display all configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Kubernetes Version: $K8S_VERSION"
echo "Network Plugin: $NETWORK_PLUGIN"
echo "Network Policy: $NETWORK_POLICY"
echo "Namespaces: $NS_FRONTEND, $NS_BACKEND, $NS_DATABASE"
echo "========================="
```

---

## 2. Create Resource Group and AKS Cluster with Network Policy

```bash
# Create resource group
RG_RESULT=$(az group create \
  --name $RESOURCE_GROUP `# Resource group name` \
  --location $LOCATION `# Azure region` \
  --query 'properties.provisioningState' \
  --output tsv)

echo "Resource Group: $RG_RESULT"

# Create AKS cluster with Azure CNI and Network Policy (7-12 minutes)
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_VM_SIZE `# Node VM size` \
  --kubernetes-version $K8S_VERSION `# K8s version` \
  --network-plugin $NETWORK_PLUGIN `# Azure CNI (required for network policies)` \
  --network-policy $NETWORK_POLICY `# Enable Azure Network Policy` \
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

# Verify cluster connectivity and network policy support
echo "Current kubectl context: $(kubectl config current-context)"
kubectl get nodes
```

---

## 3. Verify Network Policy Support

```bash
# Check Kubernetes Network Policy API availability
kubectl api-versions | grep networking.k8s.io/v1

# Verify network policy is enabled in AKS cluster
NETPOL_STATUS=$(az aks show \
  --resource-group $RESOURCE_GROUP `# Source resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --query "networkProfile.networkPolicy" \
  --output tsv)

echo "Network Policy Provider: $NETPOL_STATUS"
```

---

## 4. Create Test Namespaces

```bash
# Create namespaces for three-tier application
kubectl create namespace $NS_FRONTEND
kubectl create namespace $NS_BACKEND
kubectl create namespace $NS_DATABASE

# Verify namespace creation
kubectl get namespaces | grep -E "frontend|backend|database"
```

---

## 5. Deploy Test Applications

### Frontend Application

```bash
# Deploy frontend pod with labels
kubectl run frontend \
  --image=nginx `# Web server` \
  --labels=app=frontend,tier=frontend `# Selector labels` \
  -n $NS_FRONTEND

# Expose frontend as a service
kubectl expose pod frontend \
  --port=80 `# Service port` \
  -n $NS_FRONTEND

# Wait for pod to be ready
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/frontend \
  -n $NS_FRONTEND
```

### Backend Application

```bash
# Deploy backend pod with labels
kubectl run backend \
  --image=nginx `# Web server` \
  --labels=app=backend,tier=backend `# Selector labels` \
  -n $NS_BACKEND

# Expose backend as a service
kubectl expose pod backend \
  --port=80 `# Service port` \
  -n $NS_BACKEND

# Wait for pod to be ready
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/backend \
  -n $NS_BACKEND
```

### Database Application

```bash
# Deploy database pod with labels
kubectl run database \
  --image=postgres:15-alpine `# PostgreSQL database` \
  --labels=app=database,tier=database `# Selector labels` \
  --env="POSTGRES_PASSWORD=testpass" `# Required env var` \
  -n $NS_DATABASE

# Expose database as a service
kubectl expose pod database \
  --port=5432 `# PostgreSQL port` \
  -n $NS_DATABASE

# Wait for pod to be ready
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/database \
  -n $NS_DATABASE
```

### Verify All Deployments

```bash
# List all pods across namespaces
kubectl get pods -n $NS_FRONTEND
kubectl get pods -n $NS_BACKEND
kubectl get pods -n $NS_DATABASE

# List all services
kubectl get services -n $NS_FRONTEND
kubectl get services -n $NS_BACKEND
kubectl get services -n $NS_DATABASE
```

---

## 6. Test Pre-Policy Connectivity (Baseline - Everything Should Work)

```bash
# Test frontend → backend connectivity (HTTP)
kubectl exec -it frontend -n $NS_FRONTEND -- \
  curl -m 5 http://backend.$NS_BACKEND.svc.cluster.local

# Test frontend → database connectivity (TCP port check)
kubectl exec -it frontend -n $NS_FRONTEND -- \
  nc -zv database.$NS_DATABASE.svc.cluster.local 5432

# Test backend → database connectivity (TCP port check)
kubectl exec -it backend -n $NS_BACKEND -- \
  nc -zv database.$NS_DATABASE.svc.cluster.local 5432
```

**Expected:** All connections succeed (no network policies applied yet).

---

## 7. Apply Deny-All Ingress Policy to Database Namespace

```bash
# Create deny-all ingress policy for database namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: $NS_DATABASE
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules defined = deny all ingress traffic
EOF

# Verify policy creation
kubectl get networkpolicy -n $NS_DATABASE
```

---

## 8. Test Connectivity After Deny-All Policy

```bash
# Test frontend → database (should fail/timeout)
kubectl exec -it frontend -n $NS_FRONTEND -- \
  nc -zv -w 5 database.$NS_DATABASE.svc.cluster.local 5432

# Test backend → database (should fail/timeout)
kubectl exec -it backend -n $NS_BACKEND -- \
  nc -zv -w 5 database.$NS_DATABASE.svc.cluster.local 5432
```

**Expected:** Connection attempts timeout or fail (denied by network policy).

---

## 9. Allow Backend to Access Database

```bash
# Label backend namespace for network policy selector
kubectl label namespace $NS_BACKEND name=backend --overwrite

# Create allow policy for backend → database traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: $NS_DATABASE
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: backend
      podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
EOF

# Verify policy creation
kubectl get networkpolicy -n $NS_DATABASE
```

### Test Selective Connectivity

```bash
# Backend → database should work now
kubectl exec -it backend -n $NS_BACKEND -- \
  nc -zv database.$NS_DATABASE.svc.cluster.local 5432

# Frontend → database should still fail
kubectl exec -it frontend -n $NS_FRONTEND -- \
  nc -zv -w 5 database.$NS_DATABASE.svc.cluster.local 5432
```

**Expected:** Backend can access database, frontend cannot.

---

## 10. Apply Deny-All Ingress to All Namespaces

```bash
# Apply deny-all ingress policy to frontend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: $NS_FRONTEND
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Apply deny-all ingress policy to backend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: $NS_BACKEND
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Verify policies across all namespaces
kubectl get networkpolicies --all-namespaces
```

---

## 11. Allow Frontend to Access Backend

```bash
# Label frontend namespace for network policy selector
kubectl label namespace $NS_FRONTEND name=frontend --overwrite

# Create allow policy for frontend → backend traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: $NS_BACKEND
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
      podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

# Verify policy creation
kubectl get networkpolicy -n $NS_BACKEND
```

### Test Frontend → Backend Connectivity

```bash
# Frontend → backend should work now
kubectl exec -it frontend -n $NS_FRONTEND -- \
  curl http://backend.$NS_BACKEND.svc.cluster.local
```

**Expected:** Frontend can successfully reach backend.

---

## 12. Deny All Egress Traffic

```bash
# Apply deny-all egress policy to frontend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: $NS_FRONTEND
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny all egress traffic
EOF

# Verify policy creation
kubectl get networkpolicy -n $NS_FRONTEND
```

### Test Egress Blocking

```bash
# Frontend → backend should fail now (no egress allowed)
kubectl exec -it frontend -n $NS_FRONTEND -- \
  curl -m 5 http://backend.$NS_BACKEND.svc.cluster.local

# Frontend → internet should also fail
kubectl exec -it frontend -n $NS_FRONTEND -- \
  curl -m 5 https://www.google.com
```

**Expected:** All egress traffic blocked, including DNS resolution.

---

## 13. Allow Specific Egress (DNS and Backend)

```bash
# Create allow-egress policy for frontend (DNS + backend)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-frontend
  namespace: $NS_FRONTEND
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  # Allow DNS resolution
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # Allow traffic to backend
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend
      podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
EOF

# Verify policy creation
kubectl get networkpolicy -n $NS_FRONTEND
```

### Test Selective Egress

```bash
# Frontend → backend should work now (DNS + HTTP allowed)
kubectl exec -it frontend -n $NS_FRONTEND -- \
  curl http://backend.$NS_BACKEND.svc.cluster.local

# Frontend → internet should still fail (not in egress rules)
kubectl exec -it frontend -n $NS_FRONTEND -- \
  curl -m 5 https://www.google.com
```

**Expected:** Frontend can reach backend, but external traffic remains blocked.

---

## 14. Create Default Deny-All Template (Reusable)

```bash
# Create a reusable deny-all policy template (both ingress and egress)
cat <<'EOF' > /tmp/default-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Apply to all application namespaces
for ns in $NS_FRONTEND $NS_BACKEND $NS_DATABASE; do
  kubectl apply -f /tmp/default-deny-all.yaml -n $ns
done

# View all network policies
kubectl get networkpolicies --all-namespaces
```

---

## 15. View and Inspect Network Policies

```bash
# List all network policies across all namespaces
kubectl get networkpolicies --all-namespaces

# Describe specific policy in database namespace
kubectl describe networkpolicy deny-all-ingress -n $NS_DATABASE

# View allow-backend-to-database policy in YAML format
kubectl get networkpolicy allow-backend-to-database -n $NS_DATABASE -o yaml

# Show policy details for frontend namespace
kubectl describe networkpolicy -n $NS_FRONTEND
```

---

## 16. Allow External Egress for Specific Workloads

```bash
# Create policy allowing external HTTPS egress for backend API pods
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
  namespace: $NS_BACKEND
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Egress
  egress:
  # Allow DNS resolution
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow HTTPS to external destinations (with exceptions)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32  # Exclude Azure metadata service
        - 10.0.0.0/8          # Exclude private networks
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
EOF

# Verify policy creation
kubectl get networkpolicy -n $NS_BACKEND
```

---

## 17. Test Network Policy with Validation Pod

```bash
# Create test pod with network tools for validation
kubectl run test-network \
  --image=nicolaka/netshoot `# Network troubleshooting tools` \
  --labels=app=test,tier=frontend `# Match frontend tier` \
  -n $NS_FRONTEND \
  -- sleep 3600

# Wait for pod to be ready
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/test-network \
  -n $NS_FRONTEND

# Test backend connectivity (should work with allow policy)
kubectl exec -it test-network -n $NS_FRONTEND -- \
  curl -m 5 http://backend.$NS_BACKEND.svc.cluster.local

# Test database connectivity (should fail - not allowed)
kubectl exec -it test-network -n $NS_FRONTEND -- \
  nc -zv -w 5 database.$NS_DATABASE.svc.cluster.local 5432

# Test DNS resolution (should work if DNS egress allowed)
kubectl exec -it test-network -n $NS_FRONTEND -- \
  nslookup kubernetes.default
```

---

## Expected Results

✅ All traffic denied by default with deny-all policies  
✅ Specific allow rules enable required communication paths  
✅ Pod-to-pod isolation enforced across namespaces  
✅ DNS traffic explicitly allowed in egress policies  
✅ Backend can access database; frontend cannot  
✅ Frontend can access backend via HTTP  
✅ External traffic blocked unless explicitly allowed  
✅ Network policies are additive and namespace-scoped  

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete all test namespaces and network policies
kubectl delete namespace $NS_FRONTEND $NS_BACKEND $NS_DATABASE
```

### Option 2: Delete Entire Resource Group (Complete Cleanup)
```bash
# Delete entire resource group (removes cluster and all resources)
az group delete \
  --name $RESOURCE_GROUP `# Target resource group` \
  --yes `# Skip confirmation` \
  --no-wait `# Run asynchronously`
```

---

## Key Takeaways

- **Default-deny** is the security best practice (start with deny-all, then allowlist)
- Network policies are **additive** - multiple policies combine to allow traffic
- **Empty podSelector** `{}` matches all pods in the namespace
- **Ingress** controls incoming traffic; **Egress** controls outgoing traffic
- **DNS must be explicitly allowed** in egress policies (kube-system/kube-dns)
- Policies are **namespace-scoped** and apply to pods, not services
- **Both** source and destination rules can use pod/namespace selectors
- **IPBlock** allows CIDR-based rules for external traffic control
- Network policies apply at the **pod level**, not service level
- **Azure CNI** is required for network policies in AKS
- Policies only affect **new connections**, not established ones

---

## How This Connects to Zero-Trust Networking

This lab demonstrates **microsegmentation** in Kubernetes:

### Network Policy Enforcement Flow:

```
┌─────────────────────────────────────────┐
│  Kubernetes Network Policy              │
│  (Default-Deny + Allowlist)             │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │ 1. Deny All Ingress + Egress     │  │ ← Start here (zero-trust)
│  ├──────────────────────────────────┤  │
│  │ 2. Allow DNS (kube-system)       │  │ ← Essential for name resolution
│  ├──────────────────────────────────┤  │
│  │ 3. Allow App-to-App (Whitelist)  │  │ ← Explicit service mesh
│  ├──────────────────────────────────┤  │
│  │ 4. Allow External (Minimal)      │  │ ← Least privilege
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│     Traffic Flow Validation             │
│                                         │
│  Frontend ──✓──> Backend (HTTP:80)     │
│  Frontend ──✗──> Database (5432)       │
│  Backend  ──✓──> Database (5432)       │
│  All      ──✗──> Internet (blocked)    │
└─────────────────────────────────────────┘
```

### Enterprise Use Cases:

| Pattern | Implementation | Benefit |
|---------|---------------|---------|
| **Multi-tenancy** | Namespace isolation with deny-all | Prevent cross-tenant traffic |
| **Microservices** | Selective pod-to-pod allowlist | Service-to-service security |
| **Data tier protection** | Database namespace restrictions | Limit database access to backend only |
| **Egress control** | Block internet by default | Prevent data exfiltration |
| **Compliance** | Network segmentation logs | Audit trail for security policies |

---

## Network Policy Best Practices

| Practice | Implementation | Rationale |
|----------|---------------|-----------|
| **Start with deny-all** | Apply to all namespaces | Zero-trust foundation |
| **Explicit DNS egress** | Allow kube-system/kube-dns | Required for service discovery |
| **Label namespaces** | Use consistent naming | Enable namespace selectors |
| **Use pod labels** | tier=frontend/backend/database | Granular policy targeting |
| **CIDR exceptions** | Exclude metadata service | Prevent cloud platform access |
| **Test incrementally** | Add one allow rule at a time | Easier troubleshooting |
| **Document policies** | Annotate with business logic | Maintainability |

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| **Policies not enforced** | Network policy provider not enabled | Verify `--network-policy azure` in AKS cluster |
| **DNS resolution fails** | Missing DNS egress rule | Add explicit allow to kube-system/kube-dns on UDP:53 |
| **Connectivity timeout** | Too restrictive deny-all | Check policy order, ensure allow rules exist |
| **Existing connections work** | Policies apply to new connections | Restart pods to apply policies |
| **Wrong pod/namespace match** | Incorrect label selectors | Use `kubectl describe networkpolicy` to verify |
| **Azure CNI required** | Using kubenet | Recreate cluster with `--network-plugin azure` |

---

## Additional Resources

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Azure Network Policy Manager](https://learn.microsoft.com/azure/aks/use-network-policies)
- [AKS Network Concepts](https://learn.microsoft.com/azure/aks/concepts-network)
- [Calico Network Policy](https://docs.projectcalico.org/security/calico-network-policy)

---

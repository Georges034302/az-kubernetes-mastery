# Lab 3c: Network Policy - Deny All

## Objective
Apply deny-all policy and verify pod-to-pod isolation.

## Prerequisites
- AKS cluster with network policy enabled (Azure CNI or Calico)
- `kubectl` configured
- Understanding of Kubernetes networking

## Steps

### 1. Verify Network Policy Support
```bash
# Check if network policies are supported
kubectl api-versions | grep networking.k8s.io/v1

# For AKS, ensure network policy is enabled
az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query "networkProfile.networkPolicy" -o tsv
```

If not enabled, update cluster:
```bash
# Note: This requires Azure CNI networking
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --network-policy azure
```

### 2. Create Test Namespaces
```bash
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database
```

### 3. Deploy Test Applications

**Frontend app:**
```bash
kubectl run frontend \
  --image=nginx \
  --labels=app=frontend,tier=frontend \
  -n frontend
```

**Backend app:**
```bash
kubectl run backend \
  --image=nginx \
  --labels=app=backend,tier=backend \
  -n backend
```

**Database app:**
```bash
kubectl run database \
  --image=postgres:15-alpine \
  --labels=app=database,tier=database \
  --env="POSTGRES_PASSWORD=testpass" \
  -n database
```

Expose services:
```bash
kubectl expose pod frontend --port=80 -n frontend
kubectl expose pod backend --port=80 -n backend
kubectl expose pod database --port=5432 -n database
```

### 4. Test Connectivity Before Network Policies
```bash
# From frontend to backend (should work)
kubectl exec -it frontend -n frontend -- curl -m 5 http://backend.backend.svc.cluster.local

# From frontend to database (should work)
kubectl exec -it frontend -n frontend -- nc -zv database.database.svc.cluster.local 5432

# From backend to database (should work)
kubectl exec -it backend -n backend -- nc -zv database.database.svc.cluster.local 5432
```

**Expected:** All connections succeed (no network policies yet).

### 5. Apply Deny-All Ingress Policy to Database Namespace
Create `deny-all-ingress-database.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: database
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules defined = deny all ingress
```

Apply:
```bash
kubectl apply -f deny-all-ingress-database.yaml
```

### 6. Test Connectivity After Deny-All Policy
```bash
# From frontend to database (should fail)
kubectl exec -it frontend -n frontend -- nc -zv -w 5 database.database.svc.cluster.local 5432

# From backend to database (should fail)
kubectl exec -it backend -n backend -- nc -zv -w 5 database.database.svc.cluster.local 5432
```

**Expected:** Connection attempts timeout (denied).

### 7. Allow Backend to Access Database
Create `allow-backend-to-database.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: database
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
```

First, label the backend namespace:
```bash
kubectl label namespace backend name=backend
```

Apply the policy:
```bash
kubectl apply -f allow-backend-to-database.yaml
```

Test connectivity:
```bash
# From backend to database (should work now)
kubectl exec -it backend -n backend -- nc -zv database.database.svc.cluster.local 5432

# From frontend to database (should still fail)
kubectl exec -it frontend -n frontend -- nc -zv -w 5 database.database.svc.cluster.local 5432
```

### 8. Apply Deny-All to All Namespaces
Create `deny-all-default.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Apply:
```bash
kubectl apply -f deny-all-default.yaml
```

### 9. Create Allow Rules for Application Flow
**Allow frontend to backend:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
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
```

Label and apply:
```bash
kubectl label namespace frontend name=frontend
kubectl apply -f allow-frontend-to-backend.yaml
```

Test:
```bash
kubectl exec -it frontend -n frontend -- curl http://backend.backend.svc.cluster.local
```

### 10. Deny All Egress Traffic
Create `deny-all-egress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny all egress
```

Apply:
```bash
kubectl apply -f deny-all-egress.yaml
```

Test (should fail):
```bash
kubectl exec -it frontend -n frontend -- curl -m 5 http://backend.backend.svc.cluster.local
kubectl exec -it frontend -n frontend -- curl -m 5 https://www.google.com
```

### 11. Allow Specific Egress (DNS and Backend)
Create `allow-egress-frontend.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-frontend
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  # Allow DNS
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
  # Allow to backend
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
```

Apply:
```bash
kubectl apply -f allow-egress-frontend.yaml
```

Test:
```bash
# Should work
kubectl exec -it frontend -n frontend -- curl http://backend.backend.svc.cluster.local

# Should still fail (no external egress)
kubectl exec -it frontend -n frontend -- curl -m 5 https://www.google.com
```

### 12. Create Default Deny-All Template
Create a reusable template `default-deny-all.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Apply to multiple namespaces:
```bash
for ns in frontend backend database; do
  kubectl apply -f default-deny-all.yaml -n $ns
done
```

### 13. View Network Policies
```bash
# List all network policies
kubectl get networkpolicies --all-namespaces

# Describe specific policy
kubectl describe networkpolicy deny-all-ingress -n database

# View in YAML
kubectl get networkpolicy allow-backend-to-database -n database -o yaml
```

### 14. Test Complex Scenario
Deploy a multi-tier application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: frontend
spec:
  selector:
    app: web
  ports:
  - port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-ingress
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Allow from same namespace
    ports:
    - protocol: TCP
      port: 80
  - from:  # Allow from ingress controller
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
```

### 15. Allow External Egress for Specific Pods
Create `allow-external-egress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow HTTPS to any destination
  - to:
    - namespaceSelector: {}
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 443
  # Allow to internet (CIDR blocks)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32  # Exclude metadata service
    ports:
    - protocol: TCP
      port: 443
```

### 16. Audit Network Policy Compliance
Create a test pod for validation:

```bash
# Create test pod in frontend
kubectl run test-network \
  --image=nicolaka/netshoot \
  --labels=app=test \
  -n frontend \
  -- sleep 3600

# Test various connections
kubectl exec -it test-network -n frontend -- curl -m 5 http://backend.backend.svc.cluster.local
kubectl exec -it test-network -n frontend -- curl -m 5 http://database.database.svc.cluster.local:5432
kubectl exec -it test-network -n frontend -- nslookup kubernetes.default
```

## Expected Results
- All traffic denied by default with deny-all policies
- Specific allow rules enable required communication paths
- Pod-to-pod isolation works across namespaces
- DNS traffic must be explicitly allowed in egress policies
- Network policies are namespace-scoped
- Ingress and egress can be controlled independently
- External traffic can be restricted using CIDR blocks

## Cleanup
```bash
kubectl delete namespace frontend backend database
```

## Key Takeaways
- **Default deny** is security best practice
- Network policies are **additive** (multiple policies combine)
- **Empty podSelector** `{}` matches all pods in namespace
- **Ingress** controls incoming traffic; **Egress** controls outgoing
- DNS must be explicitly allowed in egress policies
- Policies are **namespace-scoped**
- Both source and destination rules can use pod/namespace selectors
- **IPBlock** allows CIDR-based rules for external traffic
- Policies apply at the pod level, not service level

## Network Policy Pattern: Defense in Depth

```
┌─────────────────────────────────────────┐
│  1. Deny all ingress + egress          │ ← Start here
├─────────────────────────────────────────┤
│  2. Allow DNS (kube-dns)               │ ← Essential
├─────────────────────────────────────────┤
│  3. Allow app-specific pod-to-pod      │ ← Whitelist
├─────────────────────────────────────────┤
│  4. Allow external egress (if needed)  │ ← Minimal
└─────────────────────────────────────────┘
```

## Troubleshooting
- **Policies not working**: Check if network policy provider is enabled
- **DNS not working**: Add explicit DNS egress rule to kube-system/kube-dns
- **Connectivity issues**: Use `kubectl describe networkpolicy` to verify rules
- **Conflicts**: Remember policies are additive; check all policies affecting a pod

---

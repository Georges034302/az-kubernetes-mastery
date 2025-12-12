# Lab 3b: Pod Security Admission - Deny Privileged Pods

## Objective
Enforce **Pod Security Admission (PSA)** to block privileged or insecure pods using the **Baseline** and **Restricted** Pod Security Standards. This lab demonstrates production-grade pod hardening and security policy enforcement in AKS.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Azure resource configuration
RESOURCE_GROUP="rg-aks-psa-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-psa-demo"

# AKS cluster configuration
NODE_COUNT=3
NODE_VM_SIZE="Standard_DS2_v2"
K8S_VERSION="1.28"  # PSA available in 1.23+

# Namespace configuration
NS_PRIVILEGED="psa-privileged"
NS_BASELINE="psa-baseline"
NS_RESTRICTED="psa-restricted"
NS_PRODUCTION="production"

# Display all configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Kubernetes Version: $K8S_VERSION"
echo "Namespaces: $NS_PRIVILEGED, $NS_BASELINE, $NS_RESTRICTED, $NS_PRODUCTION"
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

# Create AKS cluster (5-10 minutes)
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_VM_SIZE `# Node VM size` \
  --kubernetes-version $K8S_VERSION `# K8s version with PSA support` \
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
echo "Current kubectl context: $(kubectl config current-context)"
kubectl get nodes
```

---

## 3. Understand Pod Security Standards

Kubernetes defines three security levels:

| Standard | Description | Use Case |
|----------|-------------|----------|
| **Privileged** | Unrestricted (default) | System workloads, CNI, CSI drivers |
| **Baseline** | Minimally restrictive | General applications, prevent known escalations |
| **Restricted** | Heavily restricted | Security-critical applications, zero-trust environments |

---

## 4. Verify Pod Security Admission Availability

```bash
# Check if Pod Security admission is enabled (available in K8s 1.23+)
kubectl api-versions | grep admissionregistration

# View existing namespace labels
kubectl get namespaces --show-labels
```

---

## 5. Create Test Namespaces

```bash
# Create namespaces for testing different security levels
kubectl create namespace $NS_PRIVILEGED
kubectl create namespace $NS_BASELINE
kubectl create namespace $NS_RESTRICTED
kubectl create namespace $NS_PRODUCTION

# Verify namespace creation
kubectl get namespaces | grep -E "psa-|production"
```

---

## 6. Apply Pod Security Standards to Namespaces

### Privileged Namespace (Allow Everything)

```bash
# Apply privileged standard - allows all pod configurations
kubectl label namespace $NS_PRIVILEGED \
  pod-security.kubernetes.io/enforce=privileged `# Enforcement mode` \
  pod-security.kubernetes.io/audit=privileged `# Audit logging` \
  pod-security.kubernetes.io/warn=privileged `# Warning messages`
```

### Baseline Namespace (Block Privileged Pods)

```bash
# Apply baseline standard - prevents known privilege escalations
kubectl label namespace $NS_BASELINE \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
```

### Restricted Namespace (Strongest Controls)

```bash
# Apply restricted standard - enforces pod hardening best practices
kubectl label namespace $NS_RESTRICTED \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

### Verify Namespace Labels

```bash
# Display Pod Security labels for all test namespaces
kubectl get namespace $NS_PRIVILEGED $NS_BASELINE $NS_RESTRICTED \
  --show-labels
```

---

## 7. Test Privileged Pod in Privileged Namespace (Should Succeed)

```bash
# Create privileged pod in privileged namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: $NS_PRIVILEGED
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true  # Full host access
EOF

# Verify pod creation
kubectl get pod privileged-pod -n $NS_PRIVILEGED

# Check security context
kubectl get pod privileged-pod -n $NS_PRIVILEGED \
  -o jsonpath='{.spec.containers[0].securityContext}'
```

**Expected:** Pod created successfully with privileged: true.

---

## 8. Test Privileged Pod in Baseline Namespace (Should Fail)

```bash
# Attempt to create privileged pod in baseline namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod-blocked
  namespace: $NS_BASELINE
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
EOF
```

**Expected Error:**
```
Error from server (Forbidden): pods "privileged-pod-blocked" is forbidden: 
violates PodSecurity "baseline:latest": privileged 
(container "nginx" must not set securityContext.privileged=true)
```

---

## 9. Test Baseline-Compliant Pod

```bash
# Create baseline-compliant pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
  namespace: $NS_BASELINE
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false  # Required for baseline
EOF

# Verify pod creation
kubectl get pod baseline-pod -n $NS_BASELINE
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/baseline-pod \
  -n $NS_BASELINE
```

**Expected:** Pod created successfully.

---

## 10. Test Baseline Pod in Restricted Namespace (Should Fail)

```bash
# Attempt to create baseline pod in restricted namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod-restricted
  namespace: $NS_RESTRICTED
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
EOF
```

**Expected Error:** Fails due to missing restricted requirements (seccomp, capabilities dropped, runAsNonRoot).

---

## 11. Create Restricted-Compliant Pod

```bash
# Create fully compliant pod for restricted namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: $NS_RESTRICTED
spec:
  securityContext:
    runAsNonRoot: true  # Must not run as root
    runAsUser: 1000  # Specify non-root user
    fsGroup: 2000  # File system group
    seccompProfile:
      type: RuntimeDefault  # Required seccomp profile
  containers:
  - name: nginx
    image: nginxinc/nginx-unprivileged:alpine  # Non-root nginx image
    securityContext:
      allowPrivilegeEscalation: false  # No privilege escalation
      capabilities:
        drop:
        - ALL  # Drop all capabilities
      runAsNonRoot: true
      runAsUser: 1000
    ports:
    - containerPort: 8080
EOF

# Verify pod creation
kubectl get pod restricted-pod -n $NS_RESTRICTED
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/restricted-pod \
  -n $NS_RESTRICTED

# Verify security context
kubectl get pod restricted-pod -n $NS_RESTRICTED \
  -o jsonpath='{.spec.securityContext}' | jq
```

---

## 12. Test Violations in Restricted Namespace

### Test A: Running as Root (Should Fail)

```bash
# Attempt to run pod as root user
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: root-user-pod
  namespace: $NS_RESTRICTED
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      runAsUser: 0  # root user
EOF
```

**Expected Error:** `must set securityContext.runAsNonRoot=true`

### Test B: Missing Seccomp Profile (Should Fail)

```bash
# Attempt to create pod without seccomp profile
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-seccomp-pod
  namespace: $NS_RESTRICTED
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
  # Missing: seccompProfile at pod level
EOF
```

**Expected Error:** `seccompProfile must be set to RuntimeDefault or Localhost`

### Test C: Privilege Escalation Enabled (Should Fail)

```bash
# Attempt to enable privilege escalation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: priv-escalation-pod
  namespace: $NS_RESTRICTED
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: true  # Not allowed
      runAsNonRoot: true
EOF
```

**Expected Error:** `allowPrivilegeEscalation != false`

---

## 13. Apply Restricted Policy to Production Namespace

```bash
# Apply restricted standard to production namespace
kubectl label namespace $NS_PRODUCTION \
  pod-security.kubernetes.io/enforce=restricted `# Block non-compliant pods` \
  pod-security.kubernetes.io/audit=restricted `# Log violations` \
  pod-security.kubernetes.io/warn=restricted `# Show warnings`

# Verify production namespace labels
kubectl get namespace $NS_PRODUCTION --show-labels
```

---

## 14. Deploy Restricted-Compliant Application

```bash
# Create production deployment with full security compliance
cat <<EOF | kubectl apply -f -

apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: $NS_PRODUCTION
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
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 3000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: nginxinc/nginx-unprivileged:alpine
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
EOF

# Wait for deployment to be ready
kubectl rollout status deployment/secure-app \
  -n $NS_PRODUCTION \
  --timeout=120s

# List all pods
kubectl get pods -n $NS_PRODUCTION

# Verify security context
kubectl describe pod -n $NS_PRODUCTION -l app=secure-app | \
  grep -A 15 "Security Context"
```

---

## 15. Use Audit and Warn Modes

```bash
# Apply mixed enforcement levels - enforce baseline, warn/audit restricted
kubectl label namespace $NS_PRODUCTION \
  pod-security.kubernetes.io/enforce=baseline `# Block privileged pods` \
  pod-security.kubernetes.io/warn=restricted `# Warn on restricted violations` \
  pod-security.kubernetes.io/audit=restricted `# Audit restricted violations` \
  --overwrite

# Try deploying non-compliant pod (will succeed with warning)
kubectl run test-warn --image=nginx -n $NS_PRODUCTION

# Check warning events
kubectl get events -n $NS_PRODUCTION | grep -i warning

# Clean up test pod
kubectl delete pod test-warn -n $NS_PRODUCTION
```

---

## 16. Exempt System Namespaces

```bash
# System namespaces need privileged access for cluster operations
kubectl label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged `# Allow privileged pods` \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite

kubectl label namespace kube-public \
  pod-security.kubernetes.io/enforce=privileged \
  --overwrite

kubectl label namespace kube-node-lease \
  pod-security.kubernetes.io/enforce=privileged \
  --overwrite

# Verify system namespace labels
kubectl get namespace kube-system kube-public kube-node-lease --show-labels
```

---

## 17. View Security Violations and Audit Logs

```bash
# Get all pods with security contexts
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.securityContext}{"\n"}{end}'

# Check events for security warnings
kubectl get events --all-namespaces \
  --field-selector type=Warning | grep -i security

# View PSA labels across all namespaces
kubectl get namespaces \
  -o custom-columns=NAME:.metadata.name,ENFORCE:.metadata.labels.pod-security\\.kubernetes\\.io/enforce
```

---

## Expected Results

✅ Privileged pods blocked in baseline/restricted namespaces  
✅ Baseline policy prevents privilege escalation  
✅ Restricted policy enforces comprehensive security constraints  
✅ Compliant pods deploy successfully  
✅ Audit/warn modes provide visibility without blocking  
✅ System namespaces remain exempt from restrictions  
✅ Security contexts properly validated  

---

## How This Connects to Zero-Trust Security

This lab demonstrates **defense-in-depth** for Kubernetes workloads:

### Pod Security Admission Layers:

```
┌─────────────────────────────────────────┐
│     Kubernetes API Server               │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │  Pod Security Admission          │  │
│  │  Webhook                         │  │
│  │                                  │  │
│  │  ┌────────────────────────────┐ │  │
│  │  │ Privileged   (Allow All)   │ │  │
│  │  ├────────────────────────────┤ │  │
│  │  │ Baseline     (Block Priv)  │ │  │
│  │  ├────────────────────────────┤ │  │
│  │  │ Restricted   (Hardened)    │ │  │
│  │  └────────────────────────────┘ │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│     Pod Creation Request                │
│                                         │
│  ✓ Non-root user                       │
│  ✓ Seccomp profile                     │
│  ✓ Capabilities dropped                │
│  ✓ No privilege escalation             │
│  ✓ Read-only root filesystem           │
└─────────────────────────────────────────┘
```

### Enterprise Use Cases:

| Pattern | Implementation | Benefit |
|---------|---------------|---------|
| **Multi-tenancy** | Restricted standard per tenant namespace | Isolation and security boundaries |
| **CI/CD pipelines** | Baseline for build namespaces | Balance security and functionality |
| **Production workloads** | Restricted standard | Minimize attack surface |
| **System components** | Privileged for kube-system | Required for cluster operations |
| **Gradual rollout** | Audit/warn modes first | Identify issues before enforcing |

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete all test namespaces and workloads
kubectl delete namespace $NS_PRIVILEGED $NS_BASELINE $NS_RESTRICTED $NS_PRODUCTION
```

### Option 2: Delete Entire Resource Group (Complete Cleanup)
```bash
# Delete entire resource group (removes cluster and all resources)
DELETE_RESULT=$(az group delete \
  --name $RESOURCE_GROUP `# Target resource group` \
  --yes `# Skip confirmation` \
  --no-wait `# Run asynchronously` \
  --output tsv)

echo "Delete operation initiated: $DELETE_RESULT"

# Verify deletion status (run after a few minutes)
echo "To verify deletion status, run:"
echo "az group show --name $RESOURCE_GROUP --query properties.provisioningState"
```

---

## Key Takeaways

- **Pod Security Admission (PSA)** replaced PodSecurityPolicy (deprecated in K8s 1.21)
- **Three standards**: Privileged, Baseline, Restricted
- **Three modes**: Enforce (block), Audit (log), Warn (show warning)
- Applied via **namespace labels** (`pod-security.kubernetes.io/*`)
- **Restricted** is most secure: non-root, seccomp, drop capabilities, no privilege escalation
- Use **unprivileged container images** for restricted namespaces
- **System namespaces** must remain privileged for cluster operations
- **Audit/warn modes** help identify violations before enforcing
- PSA is **enabled by default** in Kubernetes 1.23+
- Applies to **new and updated pods** only (existing pods unaffected)

---

## Pod Security Standards Comparison

| Requirement | Privileged | Baseline | Restricted |
|------------|-----------|----------|------------|
| **Privileged containers** | ✅ Allowed | ❌ Blocked | ❌ Blocked |
| **Host namespaces** | ✅ Allowed | ❌ Blocked | ❌ Blocked |
| **Privilege escalation** | ✅ Allowed | ❌ Blocked | ❌ Blocked |
| **Root containers** | ✅ Allowed | ✅ Allowed | ❌ Blocked |
| **Seccomp profile** | ⚠️ Optional | ⚠️ Optional | ✅ Required (RuntimeDefault) |
| **Capabilities drop** | ⚠️ Optional | ⚠️ Optional | ✅ Required (ALL) |
| **Read-only root FS** | ⚠️ Optional | ⚠️ Optional | ⚠️ Recommended |
| **runAsNonRoot** | ⚠️ Optional | ⚠️ Optional | ✅ Required |

---

## Security Best Practices

| Practice | Implementation | Rationale |
|----------|---------------|-----------|
| **Use restricted for production** | Apply to production namespaces | Minimize attack surface |
| **Start with audit/warn** | Test before enforcement | Identify breaking changes |
| **Unprivileged images** | Use non-root container images | Meet restricted requirements |
| **Read-only root filesystem** | Mount emptyDir for writable paths | Immutable container filesystem |
| **Drop all capabilities** | capabilities.drop: [ALL] | Remove Linux capabilities |
| **System namespace exemption** | Keep kube-system privileged | Required for CNI, CSI, etc. |
| **Regular audits** | Check PSA violation events | Monitor compliance |

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| **Existing pods not affected** | PSA validates only new/updated pods | Recreate pods to apply PSA |
| **System pods failing** | PSA applied to system namespaces | Exempt kube-system, kube-public, kube-node-lease |
| **Custom images failing restricted** | Running as root by default | Use non-root base images or set USER in Dockerfile |
| **Read-only filesystem errors** | App writes to container filesystem | Mount emptyDir volumes for temp/cache directories |
| **Missing capabilities** | App requires specific Linux capabilities | Review if capability is truly needed, use baseline if required |
| **Seccomp profile errors** | Profile not available on nodes | Use RuntimeDefault (always available) |

---

## Additional Resources

- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [AKS Security Best Practices](https://learn.microsoft.com/azure/aks/operator-best-practices-cluster-security)
- [Kubernetes Security Guide](https://kubernetes.io/docs/concepts/security/)

---


# Lab 3b: Pod Security Admission - Deny Privileged

## Objective
Enforce restricted Pod Security admission and block privileged pods.

## Prerequisites
- AKS cluster running Kubernetes 1.23+
- `kubectl` configured with admin access
- Understanding of Pod Security Standards

## Steps

### 1. Understand Pod Security Standards
Kubernetes defines three levels:
- **Privileged**: Unrestricted (default)
- **Baseline**: Minimally restrictive, prevents known privilege escalations
- **Restricted**: Heavily restricted, follows pod hardening best practices

### 2. Check Current Pod Security Configuration
```bash
# Check if Pod Security admission is enabled
kubectl api-versions | grep admissionregistration

# View existing namespace labels
kubectl get namespaces --show-labels
```

### 3. Create Test Namespaces
```bash
kubectl create namespace psa-privileged
kubectl create namespace psa-baseline
kubectl create namespace psa-restricted
```

### 4. Apply Pod Security Standards to Namespaces

**Privileged namespace (unrestricted):**
```bash
kubectl label namespace psa-privileged \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged
```

**Baseline namespace:**
```bash
kubectl label namespace psa-baseline \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
```

**Restricted namespace:**
```bash
kubectl label namespace psa-restricted \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

Verify labels:
```bash
kubectl get ns psa-privileged psa-baseline psa-restricted --show-labels
```

### 5. Test Privileged Pod in Privileged Namespace (Should Succeed)
Create `privileged-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: psa-privileged
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
```

Apply:
```bash
kubectl apply -f privileged-pod.yaml
kubectl get pod privileged-pod -n psa-privileged
```

**Expected:** Pod created successfully.

### 6. Test Privileged Pod in Baseline Namespace (Should Fail)
```bash
kubectl apply -f privileged-pod.yaml --namespace=psa-baseline
```

**Expected output:**
```
Error from server (Forbidden): error when creating "privileged-pod.yaml": 
pods "privileged-pod" is forbidden: violates PodSecurity "baseline:latest": 
privileged (container "nginx" must not set securityContext.privileged=true)
```

### 7. Test Baseline-Compliant Pod
Create `baseline-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
  namespace: psa-baseline
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
```

Apply:
```bash
kubectl apply -f baseline-pod.yaml
kubectl get pod baseline-pod -n psa-baseline
```

**Expected:** Pod created successfully.

### 8. Test Non-Compliant Pod in Restricted Namespace
Try the baseline pod in restricted namespace:

```bash
kubectl apply -f baseline-pod.yaml --namespace=psa-restricted
```

**Expected:** Will fail because restricted requires more security constraints.

### 9. Create Restricted-Compliant Pod
Create `restricted-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: psa-restricted
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: false
      runAsNonRoot: true
      runAsUser: 1000
    ports:
    - containerPort: 8080
```

Apply:
```bash
kubectl apply -f restricted-pod.yaml
kubectl get pod restricted-pod -n psa-restricted
```

### 10. Test Violations in Restricted Namespace

**Test: Running as root**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-user-pod
  namespace: psa-restricted
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      runAsUser: 0  # root
```

```bash
kubectl apply -f root-user-pod.yaml
# Should fail: must not set runAsUser=0
```

**Test: Missing seccomp profile**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-seccomp-pod
  namespace: psa-restricted
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
```

```bash
kubectl apply -f no-seccomp-pod.yaml
# Should fail: seccompProfile must be set
```

**Test: Privilege escalation enabled**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: priv-escalation-pod
  namespace: psa-restricted
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
```

```bash
kubectl apply -f priv-escalation-pod.yaml
# Should fail: allowPrivilegeEscalation must be false
```

### 11. Apply Restricted Policy to Production Namespace
```bash
kubectl create namespace production

kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

### 12. Create Compliant Deployment
Create `secure-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
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
```

Apply:
```bash
kubectl apply -f secure-deployment.yaml
kubectl get pods -n production
kubectl describe pod -n production -l app=secure-app | grep -A 10 "Security Context"
```

### 13. Use Audit and Warn Modes
Apply different enforcement levels:

```bash
# Warn mode: Allows but warns
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted \
  --overwrite
```

Now try deploying non-compliant pod:
```bash
kubectl run test --image=nginx -n production
# Will create pod but show warning
```

Check audit logs:
```bash
kubectl get events -n production | grep -i warning
```

### 14. Exempt System Namespaces
System namespaces typically need privileged access:

```bash
# These should remain unrestricted
kubectl label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged

kubectl label namespace kube-public \
  pod-security.kubernetes.io/enforce=privileged

kubectl label namespace kube-node-lease \
  pod-security.kubernetes.io/enforce=privileged
```

### 15. Set Cluster-Wide Defaults with AdmissionConfiguration
For AKS, configure defaults using policy:

Create `psa-defaults.yaml`:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: 
      - kube-system
      - kube-public
      - kube-node-lease
```

### 16. View Violations
```bash
# Get pods with security context issues
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.securityContext}{"\n"}{end}'

# Check events for violations
kubectl get events --all-namespaces --field-selector type=Warning | grep -i security
```

### 17. Test StatefulSet with Restricted Policy
Create `statefulset-secure.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: secure-db
  namespace: production
spec:
  serviceName: secure-db
  replicas: 2
  selector:
    matchLabels:
      app: secure-db
  template:
    metadata:
      labels:
        app: secure-db
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: postgres
        image: postgres:15-alpine
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: false
          runAsNonRoot: true
          runAsUser: 999
        env:
        - name: POSTGRES_PASSWORD
          value: changeme
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f statefulset-secure.yaml
kubectl get statefulset -n production
kubectl get pods -n production -l app=secure-db
```

## Expected Results
- Privileged pods blocked in baseline/restricted namespaces
- Baseline policy prevents privilege escalation
- Restricted policy enforces comprehensive security constraints
- Compliant pods deploy successfully
- Audit/warn modes provide visibility without blocking
- System namespaces remain exempt
- StatefulSets and Deployments respect PSA policies

## Cleanup
```bash
kubectl delete namespace psa-privileged psa-baseline psa-restricted production
```

## Key Takeaways
- **Pod Security Admission (PSA)** replaced PodSecurityPolicy (deprecated)
- **Three standards**: Privileged, Baseline, Restricted
- **Three modes**: Enforce (block), Audit (log), Warn (show warning)
- Applied via namespace labels
- **Restricted** is most secure: non-root, seccomp, drop capabilities, no privilege escalation
- Use unprivileged container images for restricted namespaces
- System namespaces need privileged access
- Audit mode helps identify violations before enforcing

## Pod Security Standards Comparison

| Requirement | Baseline | Restricted |
|------------|----------|------------|
| Privileged containers | ❌ Blocked | ❌ Blocked |
| Host namespaces | ❌ Blocked | ❌ Blocked |
| Privilege escalation | ❌ Blocked | ❌ Blocked |
| Root containers | ✅ Allowed | ❌ Blocked |
| Seccomp profile | ⚠️ Optional | ✅ Required |
| Capabilities drop | ⚠️ Optional | ✅ Required (ALL) |
| Read-only root FS | ⚠️ Optional | ⚠️ Recommended |

## Troubleshooting
- **Existing pods not affected**: PSA only validates new/updated pods
- **System pods failing**: Exempt system namespaces
- **Custom images failing**: May need non-root variants
- **Read-only filesystem issues**: Mount emptyDir volumes for writable paths

---


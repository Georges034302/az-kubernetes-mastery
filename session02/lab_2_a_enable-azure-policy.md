# Lab 2a: Enable Azure Policy Add-on

## Objective
Enforce DenyPrivilegedContainer with the Azure Policy add-on.

## Prerequisites
- AKS cluster running
- Azure CLI installed and configured
- Owner or Policy Contributor role on the subscription
- `kubectl` configured

## Steps

### 1. Enable Azure Policy Add-on
Enable the Azure Policy add-on for your AKS cluster:

```bash
az aks enable-addons \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --addons azure-policy
```

Verify the add-on is enabled:
```bash
az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query "addonProfiles.azurepolicy" -o table
```

### 2. Verify Policy Controller Pods
Check that policy components are running:

```bash
kubectl get pods -n kube-system | grep azure-policy
kubectl get pods -n gatekeeper-system
```

Expected pods:
- `azure-policy-*` in `kube-system` namespace
- `gatekeeper-controller-manager-*` in `gatekeeper-system` namespace

### 3. Assign Built-in Azure Policy Initiative
Assign the "Kubernetes cluster pod security restricted standards" initiative:

```bash
# Get cluster resource ID
CLUSTER_ID=$(az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query id -o tsv)

# Assign policy initiative
az policy assignment create \
  --name "AKS-Restricted-Policy" \
  --display-name "AKS Pod Security Restricted Standards" \
  --scope $CLUSTER_ID \
  --policy-set-definition "/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d"
```

### 4. Wait for Policy Sync
Azure Policy can take 15-20 minutes to sync. Check status:

```bash
kubectl get constrainttemplates
kubectl get constraints
```

Monitor policy assignment:
```bash
az policy assignment list --scope $CLUSTER_ID -o table
```

### 5. Test with Privileged Container (Should Fail)
Create `privileged-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
```

Attempt to deploy (should be denied):
```bash
kubectl apply -f privileged-pod.yaml
```

**Expected output:**
```
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: 
[azurepolicy-k8sazurev2noprivilege-...] Privileged container is not allowed: nginx
```

### 6. Test with Non-Privileged Container (Should Succeed)
Create `non-privileged-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-privileged-pod
  namespace: default
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
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
    ports:
    - containerPort: 80
```

Deploy the compliant pod:
```bash
kubectl apply -f non-privileged-pod.yaml
kubectl get pod non-privileged-pod
```

### 7. Assign Specific "Deny Privileged Containers" Policy
For more targeted control, assign the individual policy:

```bash
az policy assignment create \
  --name "Deny-Privileged-Containers" \
  --display-name "Deny Privileged Containers in AKS" \
  --scope $CLUSTER_ID \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/95edb821-ddaf-4404-9732-666045e056b4" \
  --params '{
    "effect": {
      "value": "Deny"
    },
    "excludedNamespaces": {
      "value": ["kube-system", "gatekeeper-system", "azure-arc"]
    }
  }'
```

### 8. View Policy Compliance
Check compliance status in Azure Portal or CLI:

```bash
az policy state list \
  --resource $CLUSTER_ID \
  --filter "policyDefinitionAction eq 'deny'" \
  -o table
```

View detailed compliance:
```bash
az policy state summarize \
  --resource $CLUSTER_ID
```

### 9. Test Additional Violations
Try deploying pods that violate other policies:

**Host namespace violation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnamespace-pod
spec:
  hostNetwork: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f hostnamespace-pod.yaml
# Should be denied
```

**Host path violation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: host-volume
      mountPath: /host
  volumes:
  - name: host-volume
    hostPath:
      path: /
      type: Directory
```

```bash
kubectl apply -f hostpath-pod.yaml
# Should be denied
```

### 10. View Gatekeeper Constraints
List all active constraints:

```bash
kubectl get constraints
kubectl describe k8sazurev2noprivilege
kubectl describe k8sazurev3hostnetworkingports
```

View constraint templates:
```bash
kubectl get constrainttemplates
kubectl describe constrainttemplate k8sazurev2noprivilege
```

### 11. Monitor Policy Events
```bash
kubectl get events --all-namespaces | grep -i gatekeeper
kubectl logs -n gatekeeper-system -l control-plane=controller-manager --tail=50
```

### 12. Create Audit-Only Policy Assignment
For testing without enforcement:

```bash
az policy assignment create \
  --name "Audit-Privileged-Containers" \
  --display-name "Audit Privileged Containers" \
  --scope $CLUSTER_ID \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/95edb821-ddaf-4404-9732-666045e056b4" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

## Expected Results
- Azure Policy add-on successfully installed
- Gatekeeper components running in cluster
- Privileged containers are denied by admission webhook
- Non-privileged, compliant containers are allowed
- Policy compliance visible in Azure Portal
- Violations are logged and reportable

## Cleanup
```bash
kubectl delete pod non-privileged-pod --ignore-not-found

# Remove policy assignments
az policy assignment delete --name "Deny-Privileged-Containers" --scope $CLUSTER_ID
az policy assignment delete --name "AKS-Restricted-Policy" --scope $CLUSTER_ID

# Disable add-on (optional)
az aks disable-addons \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --addons azure-policy
```

## Key Takeaways
- **Azure Policy for AKS** enforces governance at the cluster level
- **Gatekeeper** (OPA) serves as the admission controller
- Policies can be assigned at subscription, resource group, or cluster scope
- **Deny** effect blocks non-compliant resources; **Audit** only logs violations
- Built-in policy initiatives provide comprehensive security baselines
- Excluded namespaces prevent blocking system components
- Policy sync can take 15-20 minutes initially

## Troubleshooting
- **Policies not enforcing**: Wait 15-20 minutes for sync; check `kubectl get constraints`
- **System pods failing**: Ensure system namespaces are excluded from policies
- **Assignment not visible**: Verify scope and RBAC permissions
- **Webhook timeout**: Check Gatekeeper pod logs for errors

## Additional Resources
- [Azure Policy built-in definitions for AKS](https://learn.microsoft.com/en-us/azure/aks/policy-reference)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)

---


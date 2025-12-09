# Lab 2a: Enable Azure Policy Add-on

## Objective
Enable and validate Azure Policy governance enforcement on AKS by applying built-in Pod Security policies and verifying that **privileged containers are denied** through Gatekeeper (OPA).

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Owner or Policy Contributor role on the subscription
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Azure configuration
RESOURCE_GROUP="rg-aks-policy-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-policy-demo"

# Cluster configuration
NODE_COUNT=2
NODE_VM_SIZE="Standard_DS2_v2"

# Display configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Node Count: $NODE_COUNT"
echo "Node VM Size: $NODE_VM_SIZE"
echo "========================="
```

---

## 2. Create Resource Group and AKS Cluster

```bash
# Create resource group
RG_RESULT=$(az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --query 'properties.provisioningState' \
  --output tsv)

echo "Resource Group creation: $RG_RESULT"

# Create AKS cluster
echo "Creating AKS cluster (this may take 5-10 minutes)..."
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --location $LOCATION \
  --node-count $NODE_COUNT \
  --node-vm-size $NODE_VM_SIZE \
  --enable-managed-identity \
  --generate-ssh-keys \
  --query 'provisioningState' \
  --output tsv)

echo "AKS cluster creation: $CLUSTER_RESULT"

# Get AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing

# Verify cluster connectivity
CURRENT_CONTEXT=$(kubectl config current-context)
echo "Current kubectl context: $CURRENT_CONTEXT"

# Display cluster nodes
echo "Cluster nodes:"
kubectl get nodes
```

---

## 3. Enable Azure Policy Add-on

```bash
# Enable Azure Policy add-on (installs Gatekeeper)
echo "Enabling Azure Policy add-on..."
ADDON_RESULT=$(az aks enable-addons \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --addons azure-policy \
  --query 'addonProfiles.azurepolicy.enabled' \
  --output tsv)

echo "Azure Policy add-on enabled: $ADDON_RESULT"

# Verify add-on is enabled
echo ""
echo "=== Azure Policy Add-on Status ==="
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "addonProfiles.azurepolicy" \
  --output table
```

---

## 4. Verify Policy Controller Pods

Azure Policy uses **OPA Gatekeeper** for admission control.

```bash
# Wait for pods to be ready
echo "Waiting for policy controller pods to start..."
sleep 30

# Check Azure Policy agent pods
echo ""
echo "=== Azure Policy Agent Pods ==="
kubectl get pods -n kube-system | grep azure-policy

# Check Gatekeeper controller pods
echo ""
echo "=== Gatekeeper Controller Pods ==="
kubectl get pods -n gatekeeper-system

# Verify Gatekeeper is ready
GATEKEEPER_READY=$(kubectl get pods -n gatekeeper-system \
  -l control-plane=controller-manager \
  --output jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}')

echo ""
echo "Gatekeeper controller ready: $GATEKEEPER_READY"
```

Expected pods:
- `azure-policy-*` in `kube-system` namespace
- `gatekeeper-controller-manager-*` in `gatekeeper-system` namespace

---

## 5. Assign Built-in Pod Security Restricted Initiative

```bash
# Get cluster resource ID
CLUSTER_ID=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query id \
  --output tsv)

echo "Cluster ID: $CLUSTER_ID"

# Dynamically retrieve the Pod Security Restricted initiative ID
echo ""
echo "=== Retrieving Pod Security Restricted Initiative ID ==="
RESTRICTED_INITIATIVE_ID=$(az policy set-definition list \
  --query "[?policyType=='BuiltIn' && contains(displayName, 'Kubernetes cluster pod security restricted')].id | [0]" \
  --output tsv)

echo "Initiative ID: $RESTRICTED_INITIATIVE_ID"

# Assign the restricted pod security baseline initiative
echo ""
echo "Assigning Pod Security Restricted Standards policy..."
POLICY_ASSIGNMENT=$(az policy assignment create \
  --name "AKS-Restricted-Policy" \
  --display-name "AKS Pod Security Restricted Standards" \
  --scope $CLUSTER_ID \
  --policy-set-definition "$RESTRICTED_INITIATIVE_ID" `# Dynamically retrieved initiative ID` \
  --query 'name' \
  --output tsv)

echo "Policy assignment created: $POLICY_ASSIGNMENT"
```

---

## 6. Wait for Policy Sync

Azure Policy add-on works in a multi-step sync cycle:

1. Policy assignment created in Azure
2. Azure Policy agent inside AKS pulls definitions
3. Agent generates **ConstraintTemplates**
4. Gatekeeper converts them into **Constraints**
5. Enforcement begins

⏳ **This process takes 10-20 minutes**, especially for large initiatives.

```bash
echo ""
echo "⏳ Waiting for policy sync (this takes 10-20 minutes)..."
echo "Checking policy sync status every 30 seconds..."
echo ""

# Monitor constraint templates
for i in {1..40}; do
  CONSTRAINT_COUNT=$(kubectl get constrainttemplates --no-headers 2>/dev/null | wc -l)
  TIMESTAMP=$(date '+%H:%M:%S')
  echo "[$TIMESTAMP] Attempt $i/40: $CONSTRAINT_COUNT ConstraintTemplates found"
  
  if [ "$CONSTRAINT_COUNT" -gt 0 ]; then
    echo "✓ Constraints are syncing!"
    break
  fi
  
  sleep 30
done

# Display synced constraints
echo ""
echo "=== Synced ConstraintTemplates ==="
kubectl get constrainttemplates

echo ""
echo "=== Active Constraints ==="
kubectl get constraints
```

---

## 7. Verify Policy Assignment

```bash
# List policy assignments
echo "=== Policy Assignments ==="
az policy assignment list \
  --scope $CLUSTER_ID \
  --output table

# Check compliance state (may take time to populate)
echo ""
echo "=== Policy Compliance Summary ==="
az policy state summarize \
  --resource $CLUSTER_ID \
  --output table
```

---

## 8. Test Violating Pod (Privileged Container — Should Fail)

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
      privileged: true  # Violation: privileged container
```

Attempt to deploy (should be denied):
```bash
echo "Testing privileged container (should be denied)..."
kubectl apply -f privileged-pod.yaml 2>&1 | tee /tmp/privileged-test.txt

# Check if it was denied
if grep -q "denied" /tmp/privileged-test.txt; then
  echo "✓ Privileged container was correctly denied!"
else
  echo "✗ Warning: Privileged container was not denied. Policy may still be syncing."
fi
```

**Expected error:**
```
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
[azurepolicy-k8sazurev2noprivilege-...] Privileged container is not allowed: nginx
```

---

## 9. Test Compliant Pod (Should Succeed)

Create `non-privileged-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-privileged-pod
  namespace: default
spec:
  securityContext:
    runAsNonRoot: true    # Run as non-root user
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault  # Use runtime default seccomp profile
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false  # Prevent privilege escalation
      capabilities:
        drop:
        - ALL              # Drop all capabilities
      readOnlyRootFilesystem: false
    ports:
    - containerPort: 80
```

Deploy the compliant pod:
```bash
# Deploy compliant pod
echo ""
echo "Deploying compliant pod..."
POD_RESULT=$(kubectl apply -f non-privileged-pod.yaml)
echo "$POD_RESULT"

# Wait for pod to be ready
kubectl wait --for=condition=ready \
  --timeout=60s \
  pod/non-privileged-pod 2>/dev/null || echo "Pod may take time to become ready"

# Check pod status
echo ""
echo "=== Pod Status ==="
kubectl get pod non-privileged-pod

# Verify pod is running
POD_STATUS=$(kubectl get pod non-privileged-pod \
  --output jsonpath='{.status.phase}')
echo "Pod phase: $POD_STATUS"
```

---

## 10. Assign Targeted Deny Policy (Optional)

For more granular control, assign a specific policy:

```bash
# Dynamically retrieve the privileged container policy ID
echo "=== Retrieving Privileged Container Policy ID ==="
PRIVILEGED_POLICY_ID=$(az policy definition list \
  --query "[?policyType=='BuiltIn' && contains(displayName, 'Kubernetes cluster containers should not use privileged')].id | [0]" \
  --output tsv)

echo "Policy ID: $PRIVILEGED_POLICY_ID"

echo ""
echo "Assigning targeted 'Deny Privileged Containers' policy..."
TARGET_POLICY=$(az policy assignment create \
  --name "Deny-Privileged-Containers" \
  --display-name "Deny Privileged Containers in AKS" \
  --scope $CLUSTER_ID \
  --policy "$PRIVILEGED_POLICY_ID" `# Dynamically retrieved policy ID` \
  --params '{
    "effect": {
      "value": "Deny"
    },
    "excludedNamespaces": {
      "value": ["kube-system", "gatekeeper-system", "azure-arc"]
    }
  }' \
  --query 'name' \
  --output tsv)

echo "Targeted policy assigned: $TARGET_POLICY"
```

---

## 11. Test Additional Violations

### Host namespace violation:

Create `hostnamespace-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnamespace-pod
  namespace: default
spec:
  hostNetwork: true  # Violation: use host network
  hostPID: true      # Violation: use host PID namespace
  containers:
  - name: nginx
    image: nginx
```

Test deployment:
```bash
echo "Testing host namespace violation..."
kubectl apply -f hostnamespace-pod.yaml 2>&1 | tee /tmp/hostns-test.txt

if grep -q "denied" /tmp/hostns-test.txt; then
  echo "✓ Host namespace violation correctly denied!"
else
  echo "✗ Warning: Host namespace was allowed"
fi
```

---

### HostPath violation:

Create `hostpath-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: host-volume
      mountPath: /host
  volumes:
  - name: host-volume
    hostPath:              # Violation: mount host filesystem
      path: /
      type: Directory
```

Test deployment:
```bash
echo ""
echo "Testing hostPath violation..."
kubectl apply -f hostpath-pod.yaml 2>&1 | tee /tmp/hostpath-test.txt

if grep -q "denied" /tmp/hostpath-test.txt; then
  echo "✓ HostPath violation correctly denied!"
else
  echo "✗ Warning: HostPath was allowed"
fi
```

---

## 12. Inspect Gatekeeper Constraints

```bash
# List all active constraints
echo "=== All Constraints ==="
kubectl get constraints

# Describe specific constraint (if exists)
echo ""
echo "=== Privileged Container Constraint Details ==="
kubectl describe k8sazurev2noprivilege 2>/dev/null || echo "Constraint not yet synced"

# List constraint templates
echo ""
echo "=== ConstraintTemplates ==="
kubectl get constrainttemplates

# View detailed template (if exists)
echo ""
echo "=== ConstraintTemplate Details ==="
kubectl describe constrainttemplate k8sazurev2noprivilege 2>/dev/null || echo "Template not yet synced"
```

---

## 13. Monitor Policy Events

```bash
# View Gatekeeper-related events
echo "=== Gatekeeper Events ==="
kubectl get events --all-namespaces | grep -i gatekeeper | tail -20

# Check Gatekeeper controller logs
echo ""
echo "=== Gatekeeper Controller Logs ==="
kubectl logs -n gatekeeper-system \
  -l control-plane=controller-manager \
  --tail=50

# Check Azure Policy agent logs
echo ""
echo "=== Azure Policy Agent Logs ==="
kubectl logs -n kube-system \
  -l app=azure-policy \
  --tail=50
```

---

## 14. Create Audit-Only Policy Assignment (Optional)

For testing without enforcement:

```bash
echo "Creating audit-only policy assignment..."
AUDIT_POLICY=$(az policy assignment create \
  --name "Audit-Privileged-Containers" \
  --display-name "Audit Privileged Containers" \
  --scope $CLUSTER_ID \
  --policy "$PRIVILEGED_POLICY_ID" `# Reuse dynamically retrieved policy ID` \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }' \
  --query 'name' \
  --output tsv)

echo "Audit policy assigned: $AUDIT_POLICY"
echo ""
echo "Note: Audit mode logs violations but does NOT block deployment"
```

---

## Expected Results

✅ Azure Policy add-on successfully installed  
✅ Gatekeeper components running in cluster  
✅ Privileged containers are denied by admission webhook  
✅ Non-privileged, compliant containers are allowed  
✅ Policy compliance visible in Azure Portal  
✅ Violations are logged and reportable  
✅ ConstraintTemplates and Constraints synced to cluster  

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete test pods
kubectl delete pod privileged-pod --ignore-not-found
kubectl delete pod non-privileged-pod --ignore-not-found
kubectl delete pod hostnamespace-pod --ignore-not-found
kubectl delete pod hostpath-pod --ignore-not-found

# Remove policy assignments
echo "Removing policy assignments..."
az policy assignment delete \
  --name "AKS-Restricted-Policy" \
  --scope $CLUSTER_ID

az policy assignment delete \
  --name "Deny-Privileged-Containers" \
  --scope $CLUSTER_ID \
  --only-show-errors

az policy assignment delete \
  --name "Audit-Privileged-Containers" \
  --scope $CLUSTER_ID \
  --only-show-errors

echo "Policy assignments removed"
```

### Option 2: Disable Azure Policy Add-on
```bash
# Disable add-on (removes Gatekeeper)
echo "Disabling Azure Policy add-on..."
az aks disable-addons \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --addons azure-policy

echo "Azure Policy add-on disabled"
```

### Option 3: Delete Entire Resource Group (Complete Cleanup)
```bash
# Delete entire resource group (removes all resources)
echo "Deleting resource group: $RESOURCE_GROUP..."
DELETE_RESULT=$(az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait \
  --output tsv)

echo "Delete operation initiated: $DELETE_RESULT"

# Verify deletion status (run after a few minutes)
echo ""
echo "To verify deletion status, run:"
echo "az group show --name $RESOURCE_GROUP --query properties.provisioningState"
```

---

## Key Takeaways

- **Azure Policy for AKS** enforces governance at the cluster level using Gatekeeper (OPA)
- **Gatekeeper** serves as the admission controller webhook
- Policies can be assigned at subscription, resource group, or cluster scope
- **Deny** effect blocks non-compliant resources; **Audit** only logs violations
- Built-in policy initiatives provide comprehensive security baselines
- Excluded namespaces prevent blocking system components (`kube-system`, `gatekeeper-system`)
- **Policy sync takes 10-20 minutes** initially for constraints to become active
- ConstraintTemplates define the policy logic, Constraints apply them

---

## Troubleshooting

- **Policies not enforcing** → Wait 15-20 minutes for sync; check `kubectl get constraints`
- **System pods failing** → Ensure system namespaces (`kube-system`, `gatekeeper-system`) are excluded
- **Assignment not visible** → Verify scope and RBAC permissions (Owner or Policy Contributor required)
- **Webhook timeout** → Check Gatekeeper pod logs: `kubectl logs -n gatekeeper-system -l control-plane=controller-manager`
- **Constraints not appearing** → Check Azure Policy agent logs: `kubectl logs -n kube-system -l app=azure-policy`
- **Policy still allows violations** → Constraints may still be syncing, wait longer

---

## Additional Resources

- [Azure Policy built-in definitions for AKS](https://learn.microsoft.com/en-us/azure/aks/policy-reference)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
- [Azure Policy for Kubernetes](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes)

---


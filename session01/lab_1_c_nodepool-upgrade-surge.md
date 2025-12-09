# Lab 1c: Node Pool Upgrade with Surge
<img width="1536" height="537" alt="ZIMAGE" src="https://github.com/user-attachments/assets/9b5cf7f8-1cd1-4267-a0fa-a6341353e7f0" />

## Objective
Perform a controlled node pool upgrade using **max surge** while validating application continuity and ensuring zero downtime.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Azure configuration
RESOURCE_GROUP="rg-aks-upgrade-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-upgrade-demo"

# Node pool configuration
NODEPOOL_NAME="userpool"
NODE_COUNT=2
VM_SIZE="Standard_DS2_v2"

# Application configuration
APP_NAME="ha-webapp"
REPLICAS=6
MIN_AVAILABLE=3

# Display configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Node Pool Name: $NODEPOOL_NAME"
echo "Node Count: $NODE_COUNT"
echo "VM Size: $VM_SIZE"
echo "App Replicas: $REPLICAS"
echo "Min Available (PDB): $MIN_AVAILABLE"
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

# Create AKS cluster with minimal system node pool
echo "Creating AKS cluster (this may take 5-10 minutes)..."
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --location $LOCATION \
  --node-count 1 \
  --node-vm-size Standard_DS2_v2 \
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
echo "Initial cluster nodes:"
kubectl get nodes
```

---

## 3. Create User Node Pool

```bash
# Create dedicated user node pool
echo "Creating user node pool: $NODEPOOL_NAME..."
NODEPOOL_RESULT=$(az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $VM_SIZE \
  --mode User \
  --query 'provisioningState' \
  --output tsv)

echo "Node pool creation: $NODEPOOL_RESULT"

# Verify node pool
echo ""
echo "=== Node Pools ==="
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --output table

# Display all nodes
echo ""
echo "=== All Nodes ==="
kubectl get nodes
```

---

## 4. Check Kubernetes Version

```bash
# Get current cluster version
CURRENT_VERSION=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "kubernetesVersion" \
  --output tsv)

echo "Current Kubernetes version: $CURRENT_VERSION"

# List available upgrade versions
echo ""
echo "=== Available Upgrades ==="
az aks get-upgrades \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --output table

# Check node pool versions
echo ""
echo "=== Node Pool Versions ==="
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --query "[].{Name:name, Version:orchestratorVersion, VmSize:vmSize}" \
  --output table
```

---

## 5. Deploy HA Test Application

Create `ha-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-webapp
  namespace: default
spec:
  replicas: 6                # Multiple replicas for HA
  selector:
    matchLabels:
      app: ha-webapp
  template:
    metadata:
      labels:
        app: ha-webapp
    spec:
      affinity:
        podAntiAffinity:     # Spread pods across nodes
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: ha-webapp
              topologyKey: kubernetes.io/hostname
      containers:
      - name: webapp
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        livenessProbe:       # Detect unhealthy containers
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:      # Control when pod receives traffic
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: ha-webapp-svc
spec:
  selector:
    app: ha-webapp
  type: LoadBalancer       # External access
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ha-webapp-pdb
spec:
  minAvailable: 3          # At least 3 pods must remain available during disruptions
  selector:
    matchLabels:
      app: ha-webapp
```

Apply the deployment:
```bash
# Deploy the application
DEPLOY_RESULT=$(kubectl apply -f ha-deployment.yaml)
echo "$DEPLOY_RESULT"

# Wait for deployment to be ready
echo "Waiting for deployment to be ready..."
kubectl wait --for=condition=available \
  --timeout=120s \
  deployment/$APP_NAME

# Check pod distribution across nodes
echo ""
echo "=== Pod Distribution ==="
kubectl get pods -l app=$APP_NAME -o wide

# Get service external IP (may take a minute)
echo ""
echo "Waiting for LoadBalancer IP..."
kubectl get svc ha-webapp-svc

# Store external IP
EXTERNAL_IP=$(kubectl get svc ha-webapp-svc \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

if [ -n "$EXTERNAL_IP" ]; then
  echo "External IP: $EXTERNAL_IP"
  echo "Test URL: http://$EXTERNAL_IP"
else
  echo "LoadBalancer IP not yet assigned, waiting..."
  sleep 30
  EXTERNAL_IP=$(kubectl get svc ha-webapp-svc \
    --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo "External IP: $EXTERNAL_IP"
fi

# Verify PDB
echo ""
echo "=== PodDisruptionBudget Status ==="
kubectl get pdb ha-webapp-pdb
```

---

## 6. Monitor Application During Upgrade

**Terminal 1 - Monitor Pods:**
```bash
# Set variable if in new terminal
APP_NAME="ha-webapp"

# Watch pod status continuously
kubectl get pods -l app=$APP_NAME -w
```

**Terminal 2 - Test Availability:**
```bash
# Set variable if in new terminal
EXTERNAL_IP=$(kubectl get svc ha-webapp-svc \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Testing availability at: http://$EXTERNAL_IP"
echo "Press Ctrl+C to stop"
echo ""

# Continuous availability test
while true; do 
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://$EXTERNAL_IP)
  TIMESTAMP=$(date '+%H:%M:%S')
  
  if [ "$HTTP_CODE" == "200" ]; then
    echo "[$TIMESTAMP] ✓ HTTP $HTTP_CODE - OK"
  else
    echo "[$TIMESTAMP] ✗ HTTP $HTTP_CODE - FAILED"
  fi
  
  sleep 1
done
```

---

## 7. Check and Configure Max Surge

```bash
# Check current max-surge setting
CURRENT_SURGE=$(az aks nodepool show \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --query "upgradeSettings.maxSurge" \
  --output tsv)

echo "Current max-surge: $CURRENT_SURGE"

# Set max surge for safer upgrades
SURGE_VALUE="33%"  # Options: 1, 33%, 50%, 100%

echo "Setting max-surge to: $SURGE_VALUE"
SURGE_RESULT=$(az aks nodepool update \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --max-surge $SURGE_VALUE \
  --query 'upgradeSettings.maxSurge' \
  --output tsv)

echo "Max-surge updated to: $SURGE_RESULT"
```

**Max Surge Options:**
- `1` or `33%`: One extra node at a time (slower, more conservative, lower cost)
- `2` or `50%`: Two extra nodes (faster, balanced)
- `100%`: Double the nodes (fastest, highest cost during upgrade)

---

## 8. Upgrade Node Pool

```bash
# Determine target version (use one from available upgrades)
# Example: If current is 1.27.x, target might be 1.28.x
TARGET_VERSION="1.28.5"  # Replace with available version from step 4

echo "Upgrading node pool '$NODEPOOL_NAME' from $CURRENT_VERSION to $TARGET_VERSION"
echo "This will take approximately 10-15 minutes..."
echo ""

# Start upgrade with max-surge
az aks nodepool upgrade \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --kubernetes-version $TARGET_VERSION \
  --max-surge $SURGE_VALUE \
  --no-wait

echo "Upgrade initiated in background"
```

---

## 9. Observe Surge Behavior

```bash
# Watch node status in real-time
echo "=== Watching Node Changes (Ctrl+C to stop) ==="
kubectl get nodes -l agentpool=$NODEPOOL_NAME -w
```

In a separate terminal:
```bash
# Set variables if needed
NODEPOOL_NAME="userpool"

# Count nodes during upgrade
while true; do
  NODE_COUNT=$(kubectl get nodes -l agentpool=$NODEPOOL_NAME \
    --no-headers 2>/dev/null | wc -l)
  TIMESTAMP=$(date '+%H:%M:%S')
  echo "[$TIMESTAMP] Nodes in pool: $NODE_COUNT"
  sleep 5
done
```

**Expected behavior:**
1. **Surge nodes added** - Extra nodes appear (e.g., 2 → 3 if surge=33%)
2. **Old nodes cordoned** - Marked as `SchedulingDisabled`
3. **Pods drained** - Moved to new nodes gracefully
4. **Old nodes deleted** - Removed one by one
5. **Process repeats** - Until all nodes upgraded

---

## 10. Verify Upgrade Success

```bash
# Wait for upgrade to complete
echo "Checking upgrade status..."
UPGRADE_STATE=$(az aks nodepool show \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --query "provisioningState" \
  --output tsv)

echo "Node pool state: $UPGRADE_STATE"

# Confirm new Kubernetes version
NEW_VERSION=$(az aks nodepool show \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --query "orchestratorVersion" \
  --output tsv)

echo "Node pool version: $NEW_VERSION"

# Check all nodes are on new version
echo ""
echo "=== Node Versions ==="
kubectl get nodes -l agentpool=$NODEPOOL_NAME \
  -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion,STATUS:.status.conditions[-1].type

# Verify application is still healthy
echo ""
echo "=== Application Status ==="
kubectl get deployment $APP_NAME
kubectl get pods -l app=$APP_NAME -o wide

# Check PDB was respected
echo ""
echo "=== PDB Status ==="
kubectl get pdb ha-webapp-pdb

# Test external access
echo ""
EXTERNAL_IP=$(kubectl get svc ha-webapp-svc \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Testing external access..."
curl -I http://$EXTERNAL_IP
```

---

## 📘 How This Connects to Cluster Autoscaler

Surge upgrades and Cluster Autoscaler serve **different purposes** but work together in production AKS environments:

### Key Differences:

| Feature | Surge Upgrade | Cluster Autoscaler |
|---------|---------------|-------------------|
| **Purpose** | Safe node replacement during upgrades | Scale nodes based on workload demand |
| **Trigger** | Manual upgrade operation | Pending pods / unused nodes |
| **Duration** | Temporary (during upgrade only) | Continuous monitoring |
| **Node behavior** | Add → drain → delete (rolling) | Add when needed, remove when idle |
| **Control** | AKS upgrade engine | CA watches pod resource requests |

### How They Work Together:

**During Normal Operations:**
- Cluster Autoscaler adds/removes nodes based on pod demand
- Node pools scale between `min-count` and `max-count`

**During Upgrades:**
- Surge mechanism temporarily **overrides** autoscaler limits
- Extra nodes created for safe pod migration
- After upgrade, autoscaler resumes normal operation

**Example Scenario:**
1. Node pool has 3 nodes (min=2, max=5)
2. Cluster Autoscaler is actively managing this pool
3. Admin triggers upgrade with `max-surge=33%`
4. Surge adds 1 extra node (total 4) **even if autoscaler doesn't need it**
5. Upgrade completes, surge node removed
6. Autoscaler resumes managing 2-5 nodes based on demand

### Why Both Matter:

- **Surge** = **Safety during maintenance** (zero-downtime upgrades)
- **Autoscaler** = **Cost optimization during runtime** (scale to demand)
- **Together** = **Production-grade AKS** (safe operations + efficient scaling)

**Next lab (if included)** will show how Cluster Autoscaler responds to actual workload changes by adding/removing nodes automatically.

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete application resources
kubectl delete deployment $APP_NAME
kubectl delete service ha-webapp-svc
kubectl delete pdb ha-webapp-pdb

# Verify cleanup
echo "Remaining resources:"
kubectl get all -l app=$APP_NAME
```

### Option 2: Delete Node Pool
```bash
# Delete the user node pool
echo "Deleting node pool: $NODEPOOL_NAME..."
az aks nodepool delete \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --no-wait

echo "Node pool deletion initiated"
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

## Expected Results

✅ Node pool successfully upgraded with zero or minimal downtime  
✅ Application remained accessible throughout upgrade (HTTP 200 responses)  
✅ PodDisruptionBudget prevented excessive pod disruptions  
✅ Max surge temporarily increased node count during upgrade  
✅ All pods rescheduled successfully to new nodes  
✅ Pod anti-affinity maintained distribution across nodes  

---

## Key Takeaways

- **Max surge** controls how many extra nodes are created during upgrades for safe pod migration
- Higher surge = faster upgrades but higher temporary costs
- **PodDisruptionBudgets (PDB)** ensure minimum pod availability during disruptions
- **Pod anti-affinity** spreads pods across nodes for better HA
- Node upgrades follow: cordon → drain → delete → replace pattern
- Proper health checks (liveness/readiness) ensure smooth pod migrations
- Surge upgrades are orchestrated by AKS, not Cluster Autoscaler

---

## Troubleshooting

- **Upgrade stuck** → Check PDB constraints, may be blocking drains
- **Pods not rescheduling** → Verify resource availability on new nodes
- **Downtime occurred** → Increase replica count or adjust PDB `minAvailable`
- **Upgrade failed** → Check Activity Logs: `az monitor activity-log list`
- **Surge nodes not appearing** → Verify `--max-surge` was set correctly

---



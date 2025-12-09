# Lab 2b: KEDA Queue Autoscaling

## Objective
Deploy **KEDA** (Kubernetes Event-Driven Autoscaling) on AKS, configure a **ScaledObject** that monitors an Azure Storage Queue, and observe **event-driven autoscaling** from 0 → N replicas based on queue depth, then back to 0 when empty.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Helm 3 installed
- Sufficient Azure subscription quota for AKS and Storage

---

## 1. Set Lab Parameters

```bash
# Azure configuration
RESOURCE_GROUP="rg-aks-keda-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-keda-demo"

# Cluster configuration
NODE_COUNT=2
NODE_VM_SIZE="Standard_DS2_v2"

# Storage configuration
STORAGE_ACCOUNT="keda$(openssl rand -hex 6)"
QUEUE_NAME="workitems"

# KEDA configuration
KEDA_NAMESPACE="keda"
KEDA_VERSION="2.12.0"
APP_NAMESPACE="default"

# Display configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Storage Account: $STORAGE_ACCOUNT"
echo "Queue Name: $QUEUE_NAME"
echo "KEDA Version: $KEDA_VERSION"
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
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_VM_SIZE `# Node VM size` \
  --enable-managed-identity `# Use managed identity` \
  --generate-ssh-keys `# Auto-generate SSH keys` \
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

## 3. Create Azure Storage Account and Queue

```bash
# Create storage account
echo "Creating Azure Storage Account..."
STORAGE_RESULT=$(az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS `# Locally redundant storage` \
  --kind StorageV2 `# General-purpose v2` \
  --query 'provisioningState' \
  --output tsv)

echo "Storage Account creation: $STORAGE_RESULT"

# Get connection string
echo ""
echo "=== Retrieving Storage Connection String ==="
CONNECTION_STRING=$(az storage account show-connection-string \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query 'connectionString' \
  --output tsv)

echo "Connection string retrieved (length: ${#CONNECTION_STRING} chars)"

# Create queue
echo ""
echo "Creating queue: $QUEUE_NAME..."
QUEUE_RESULT=$(az storage queue create \
  --name $QUEUE_NAME \
  --connection-string "$CONNECTION_STRING" \
  --query 'created' \
  --output tsv)

echo "Queue created: $QUEUE_RESULT"

# Verify queue exists
echo ""
echo "=== Storage Queue Status ==="
az storage queue show \
  --name $QUEUE_NAME \
  --connection-string "$CONNECTION_STRING" \
  --query '{Name:name, MessageCount:approximateMessageCount}' \
  --output table
```

---

## 4. Install KEDA Using Helm

```bash
# Add KEDA Helm repository
echo "Adding KEDA Helm repository..."
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# Create KEDA namespace
kubectl create namespace $KEDA_NAMESPACE

# Install KEDA
echo ""
echo "Installing KEDA version $KEDA_VERSION..."
KEDA_INSTALL=$(helm install keda kedacore/keda \
  --namespace $KEDA_NAMESPACE \
  --version $KEDA_VERSION \
  --wait \
  --timeout 5m \
  --output json | jq -r '.info.status')

echo "KEDA installation status: $KEDA_INSTALL"

# Wait for KEDA pods to be ready
echo ""
echo "Waiting for KEDA pods to be ready..."
kubectl wait --for=condition=ready \
  --timeout=120s \
  pod -l app.kubernetes.io/name=keda \
  -n $KEDA_NAMESPACE

# Verify KEDA installation
echo ""
echo "=== KEDA Pods ==="
kubectl get pods -n $KEDA_NAMESPACE

# Verify KEDA CRDs
echo ""
echo "=== KEDA Custom Resource Definitions ==="
kubectl get crd | grep keda
```

Expected CRDs:
- `scaledobjects.keda.sh`
- `scaledjobs.keda.sh`
- `triggerauthentications.keda.sh`
- `clustertriggerauthentications.keda.sh`

---

---

## 5. Create Kubernetes Secret for Storage Connection String

```bash
# Create secret with connection string
echo "Creating Kubernetes secret for Azure Storage..."
kubectl create secret generic azure-storage-secret \
  --from-literal=connection-string="$CONNECTION_STRING" \
  --namespace $APP_NAMESPACE

# Verify secret creation
SECRET_CREATED=$(kubectl get secret azure-storage-secret \
  -n $APP_NAMESPACE \
  --output jsonpath='{.metadata.name}')

echo "Secret created: $SECRET_CREATED"
```

---

## 6. Deploy Queue Consumer Application

Create `queue-consumer-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue-consumer
  namespace: default
spec:
  replicas: 0  # KEDA will manage replicas (starts at 0)
  selector:
    matchLabels:
      app: queue-consumer
  template:
    metadata:
      labels:
        app: queue-consumer
    spec:
      containers:
      - name: consumer
        image: mcr.microsoft.com/azure-cli
        command:
        - /bin/bash
        - -c
        - |
          while true; do
            echo "[$(date '+%Y-%m-%d %H:%M:%S')] Checking queue..."
            
            # Get first message from queue
            MESSAGE=$(az storage message get \
              --queue-name "${QUEUE_NAME}" \
              --connection-string "${CONNECTION_STRING}" \
              --query '[0].content' \
              --output tsv 2>/dev/null)
            
            if [ ! -z "$MESSAGE" ] && [ "$MESSAGE" != "null" ]; then
              echo "Processing message: $MESSAGE"
              
              # Simulate processing time
              sleep 10
              
              # Get message ID and pop receipt for deletion
              MSG_ID=$(az storage message get \
                --queue-name "${QUEUE_NAME}" \
                --connection-string "${CONNECTION_STRING}" \
                --query '[0].id' \
                --output tsv 2>/dev/null)
              
              POP_RECEIPT=$(az storage message get \
                --queue-name "${QUEUE_NAME}" \
                --connection-string "${CONNECTION_STRING}" \
                --query '[0].popReceipt' \
                --output tsv 2>/dev/null)
              
              # Delete processed message
              az storage message delete \
                --id "$MSG_ID" \
                --pop-receipt "$POP_RECEIPT" \
                --queue-name "${QUEUE_NAME}" \
                --connection-string "${CONNECTION_STRING}" 2>/dev/null
              
              echo "Message processed and deleted"
            else
              echo "No messages in queue"
              sleep 5
            fi
          done
        env:
        - name: QUEUE_NAME
          value: "workitems"
        - name: CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: azure-storage-secret
              key: connection-string
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

Deploy the application:
```bash
# Apply deployment manifest
echo "Deploying queue consumer application..."
kubectl apply -f queue-consumer-deployment.yaml

# Verify deployment created
DEPLOYMENT_CREATED=$(kubectl get deployment queue-consumer \
  --output jsonpath='{.metadata.name}')

echo "Deployment created: $DEPLOYMENT_CREATED"

# Check initial replica count (should be 0)
REPLICA_COUNT=$(kubectl get deployment queue-consumer \
  --output jsonpath='{.spec.replicas}')

echo "Initial replica count: $REPLICA_COUNT"
```

---

## 7. Create KEDA ScaledObject and TriggerAuthentication

Create `keda-scaledobject.yaml`:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-consumer-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: queue-consumer  # Target deployment
  minReplicaCount: 0      # Scale down to zero when queue empty
  maxReplicaCount: 10     # Maximum replicas during peak load
  pollingInterval: 10     # Check queue every 10 seconds
  cooldownPeriod: 30      # Wait 30 seconds before scaling down
  triggers:
  - type: azure-queue
    metadata:
      queueName: workitems
      queueLength: "5"    # Scale up when >5 messages per pod
      connectionFromEnv: CONNECTION_STRING
    authenticationRef:
      name: azure-queue-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-queue-auth
  namespace: default
spec:
  secretTargetRef:
  - parameter: connection
    name: azure-storage-secret
    key: connection-string
```

Apply the ScaledObject:
```bash
# Apply KEDA ScaledObject and TriggerAuthentication
echo "Creating KEDA ScaledObject..."
kubectl apply -f keda-scaledobject.yaml

# Wait for ScaledObject to be ready
echo ""
echo "Waiting for ScaledObject to initialize..."
sleep 5

# Verify ScaledObject creation
echo ""
echo "=== KEDA ScaledObject ==="
kubectl get scaledobject

# Describe ScaledObject for details
echo ""
echo "=== ScaledObject Details ==="
kubectl describe scaledobject queue-consumer-scaler

# Verify TriggerAuthentication
echo ""
echo "=== TriggerAuthentication ==="
kubectl get triggerauthentication
```

---

## 8. Verify Initial State (Zero Replicas)

```bash
# Check deployment replica count
echo "=== Current Deployment State ==="
kubectl get deployment queue-consumer

# Check pods (should be none)
echo ""
echo "=== Current Pods ==="
kubectl get pods -l app=queue-consumer

# Check HPA created by KEDA
echo ""
echo "=== KEDA-Generated HPA ==="
kubectl get hpa
```

Expected output:
- Deployment replicas: **0/0**
- Pods: **No resources found**
- HPA created by KEDA with external metrics

---

## 9. Add Messages to Queue (Trigger Scaling)

```bash
# Add messages to trigger autoscaling
echo "Adding messages to queue..."
for i in {1..20}; do
  az storage message put \
    --queue-name $QUEUE_NAME \
    --content "Task-$i-$(date +%s)" \
    --connection-string "$CONNECTION_STRING" \
    --output none
  echo "Added message $i/20"
done

# Check queue length
echo ""
echo "=== Queue Status ==="
QUEUE_LENGTH=$(az storage queue show \
  --name $QUEUE_NAME \
  --connection-string "$CONNECTION_STRING" \
  --query 'approximateMessageCount' \
  --output tsv)

echo "Approximate message count: $QUEUE_LENGTH"
```

---

## 10. Observe KEDA Autoscaling (0 → N)

```bash
# Watch deployment scale up
echo ""
echo "=== Watching Deployment Scale Up ==="
echo "Press Ctrl+C to stop watching"
echo ""
kubectl get deployment queue-consumer -w
```

In a separate terminal, monitor pods:
```bash
# Watch pods being created
kubectl get pods -l app=queue-consumer -w
```

And monitor the HPA:
```bash
# Check HPA metrics
kubectl get hpa -w
```

Expected behavior:
- KEDA detects queue messages
- Deployment scales from **0 → N** replicas
- Pods start processing messages
- HPA shows external metric from KEDA

---

## 11. Monitor Queue Processing

```bash
# View consumer logs (all pods)
kubectl logs -l app=queue-consumer \
  -f \
  --max-log-requests=10 \
  --prefix=true

# Or view logs from specific pod
POD_NAME=$(kubectl get pods -l app=queue-consumer \
  --output jsonpath='{.items[0].metadata.name}')

kubectl logs $POD_NAME -f
```

Expected log output:
```
[2025-12-09 10:15:23] Checking queue...
Processing message: Task-1-1733742923
Message processed and deleted
```

---

## 12. Observe Scale Down to Zero

Once the queue is empty, KEDA will scale down:

```bash
# Monitor queue status
echo "Monitoring queue and deployment status..."
while true; do
  QUEUE_COUNT=$(az storage queue show \
    --name $QUEUE_NAME \
    --connection-string "$CONNECTION_STRING" \
    --query 'approximateMessageCount' \
    --output tsv 2>/dev/null)
  
  REPLICA_COUNT=$(kubectl get deployment queue-consumer \
    --output jsonpath='{.spec.replicas}')
  
  TIMESTAMP=$(date '+%H:%M:%S')
  echo "[$TIMESTAMP] Queue: $QUEUE_COUNT messages | Replicas: $REPLICA_COUNT"
  
  sleep 5
done
```

Expected behavior:
- Queue empties as messages are processed
- After cooldown period (30s), deployment scales down
- Final state: **0 replicas** (scale to zero)

---

## 13. Test Scale-Up Again (Optional)

```bash
# Add more messages to trigger another scale-up
echo "Adding 10 more messages..."
for i in {21..30}; do
  az storage message put \
    --queue-name $QUEUE_NAME \
    --content "Task-$i-$(date +%s)" \
    --connection-string "$CONNECTION_STRING" \
    --output none
done

# Watch deployment scale up again
kubectl get deployment queue-consumer -w
```

---

## 14. Advanced: Modify Scaling Threshold (Optional)

```bash
# Patch ScaledObject to change queue length threshold
echo "Updating queueLength threshold to 2..."
kubectl patch scaledobject queue-consumer-scaler \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/triggers/0/metadata/queueLength", "value": "2"}]'

# Verify the change
kubectl get scaledobject queue-consumer-scaler \
  --output jsonpath='{.spec.triggers[0].metadata.queueLength}'
echo ""

# Add messages to test new threshold
echo "Adding messages to test new threshold..."
for i in {31..40}; do
  az storage message put \
    --queue-name $QUEUE_NAME \
    --content "HighPriority-Task-$i-$(date +%s)" \
    --connection-string "$CONNECTION_STRING" \
    --output none
done

# Observe faster scaling with lower threshold
kubectl get deployment queue-consumer -w
```

---

## 15. View KEDA External Metrics

```bash
# List all external metrics available
echo "=== Available External Metrics ==="
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1 | jq -r '.resources[].name'

# Get specific KEDA metric for our queue
echo ""
echo "=== KEDA Queue Metric ==="
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/default/s0-azure-queue-workitems" 2>/dev/null | jq . || echo "Metric not yet available"

# Check KEDA operator logs
echo ""
echo "=== KEDA Operator Logs ==="
kubectl logs -n $KEDA_NAMESPACE \
  -l app.kubernetes.io/name=keda-operator \
  --tail=20
```

---

## Expected Results

✅ KEDA successfully installed with all components running  
✅ Storage queue created and accessible  
✅ Deployment scales from **0 → N** based on queue depth  
✅ Messages processed and deleted automatically  
✅ Deployment scales back to **0** when queue empties  
✅ ScaledObject and TriggerAuthentication configured correctly  
✅ External metrics provider operational  
✅ Event-driven autoscaling responds within polling interval  

---

## How This Connects to Cluster Autoscaler

KEDA provides **pod-level event-driven scaling**, while the Cluster Autoscaler handles **node-level scaling**.

### Integration Flow:

```
Queue Messages Arrive
    ↓
KEDA Detects Queue Depth
    ↓
ScaledObject Scales Deployment (0 → N pods)
    ↓
Scheduler Attempts Pod Placement
    ↓
If No Available Nodes → Pods in "Pending" State
    ↓
Cluster Autoscaler Detects Pending Pods
    ↓
CA Provisions New Nodes
    ↓
Pods Scheduled on New Nodes
    ↓
Workload Processes Queue Messages
    ↓
Queue Empties
    ↓
KEDA Scales Pods Down (N → 0)
    ↓
Cluster Autoscaler Removes Unused Nodes
```

### Combined Benefits:

- **KEDA:** Enables scale-to-zero for cost optimization during idle periods
- **Cluster Autoscaler:** Provides node capacity for KEDA-scaled workloads
- **Together:** Complete event-driven infrastructure that scales both pods AND nodes

### Comparison to Session 1:

| Session 1 (HPA) | Session 2 (KEDA) |
|----------------|------------------|
| CPU/Memory-based | Event-driven (queue depth, messages, etc.) |
| Min replicas: 1 | Min replicas: **0** (scale to zero) |
| Built-in metrics | 50+ external scalers |
| Pod autoscaling only | Pods + Jobs |

This demonstrates **advanced autoscaling pipelines** used in real enterprise workloads where both resource-based and event-driven scaling work together.

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete KEDA resources
kubectl delete scaledobject queue-consumer-scaler
kubectl delete triggerauthentication azure-queue-auth

# Delete application resources
kubectl delete deployment queue-consumer
kubectl delete secret azure-storage-secret

echo "Kubernetes resources deleted"
```

### Option 2: Delete Storage Resources
```bash
# Delete queue
az storage queue delete \
  --name $QUEUE_NAME \
  --connection-string "$CONNECTION_STRING"

# Delete storage account
az storage account delete \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --yes

echo "Storage resources deleted"
```

### Option 3: Uninstall KEDA
```bash
# Uninstall KEDA Helm release
helm uninstall keda -n $KEDA_NAMESPACE

# Delete KEDA namespace
kubectl delete namespace $KEDA_NAMESPACE

echo "KEDA uninstalled"
```

### Option 4: Delete Entire Resource Group (Complete Cleanup)
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

- **KEDA** enables event-driven autoscaling based on external metrics (queue depth, messages, etc.)
- Supports **scale-to-zero** for maximum cost efficiency during idle periods
- **ScaledObject** defines scaling behavior, polling intervals, and triggers
- **TriggerAuthentication** securely passes credentials from Kubernetes secrets
- Works with **50+ scalers**: Azure Queue, Service Bus, Kafka, Redis, Prometheus, HTTP, and more
- More efficient than traditional HPA for event-driven and asynchronous workloads
- **ScaledJob** creates Kubernetes Jobs instead of managing Deployment replicas (batch processing)
- Integrates seamlessly with Cluster Autoscaler for complete infrastructure autoscaling

---

## Comparison: KEDA vs HPA

| Feature | KEDA | HPA |
|---------|------|-----|
| **Scale to zero** | ✅ Yes (min 0) | ❌ No (min 1) |
| **External metrics** | ✅ 50+ scalers | ⚠️ Limited (custom metrics API) |
| **Event-driven** | ✅ Native support | ❌ Polling metrics only |
| **Jobs support** | ✅ ScaledJob | ❌ Deployments only |
| **Use case** | Event/message-driven | Resource-based (CPU/memory) |
| **Trigger sources** | Queues, topics, HTTP, DB | CPU, memory, custom |

---

## Troubleshooting

- **Pods not scaling** → Check ScaledObject status: `kubectl describe scaledobject queue-consumer-scaler`
- **Authentication errors** → Verify secret exists and TriggerAuthentication references correct key
- **Queue not detected** → Ensure connection string is valid; check KEDA operator logs
- **Slow scaling** → Adjust `pollingInterval` (default 30s, can be reduced to 5-10s)
- **External metrics not available** → Wait 30-60s for KEDA metrics server to sync
- **Scale to zero not working** → Verify `minReplicaCount: 0` in ScaledObject spec
- **KEDA pods not running** → Check Helm installation: `helm list -n keda`

---

## Additional Resources

- [KEDA Documentation](https://keda.sh/docs/)
- [Azure Queue Scaler](https://keda.sh/docs/scalers/azure-storage-queue/)
- [KEDA Scalers List](https://keda.sh/docs/scalers/)
- [ScaledObject Specification](https://keda.sh/docs/concepts/scaling-deployments/)
- [ScaledJob Specification](https://keda.sh/docs/concepts/scaling-jobs/)

---


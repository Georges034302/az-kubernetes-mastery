# Lab 1b: Create a User Node Pool for Batch Workloads
<img width="1536" height="1024" alt="ZIMAGE" src="https://github.com/user-attachments/assets/0d9d5fb4-d407-4c39-955c-4b82c225a04f" />

## Objective
Create a dedicated AKS user node pool for batch workloads using labels, taints, and tolerations, then validate workload placement through Jobs, DaemonSets, and node affinity.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Azure configuration
RESOURCE_GROUP="rg-aks-nodepool-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-nodepool-demo"

# Node pool configuration
NODEPOOL_NAME="batchpool"
NODE_COUNT=2
VM_SIZE="Standard_D4s_v3"
MIN_COUNT=1
MAX_COUNT=5

# Display configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Node Pool Name: $NODEPOOL_NAME"
echo "Node Count: $NODE_COUNT"
echo "VM Size: $VM_SIZE"
echo "Autoscaler Range: $MIN_COUNT-$MAX_COUNT"
echo "========================="
```

---

## 2. Create Resource Group and AKS Cluster

```bash
# Create resource group and capture result
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
---

## 3. Create User Node Pool

```bash
# Create dedicated node pool for batch workloads
echo "Creating user node pool: $NODEPOOL_NAME..."
NODEPOOL_RESULT=$(az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $VM_SIZE \
  --mode User \
  --labels workload=batch environment=production \
  --node-taints workload=batch:NoSchedule \
  --enable-cluster-autoscaler \
  --min-count $MIN_COUNT \
  --max-count $MAX_COUNT \
  --query 'provisioningState' \
  --output tsv)

echo "Node pool creation: $NODEPOOL_RESULT"
```

---

## 4. Verify Node Pool and Labels

```bash
# List all node pools
echo "=== Node Pools ==="
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --output table

# Get node pool details
NODEPOOL_COUNT=$(az aks nodepool show \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --query 'count' \
  --output tsv)

echo ""
echo "Node pool '$NODEPOOL_NAME' has $NODEPOOL_COUNT nodes"

# Inspect node labels
echo ""
echo "=== Node Labels ==="
kubectl get nodes --show-labels | grep $NODEPOOL_NAME

# Inspect node taints
echo ""
echo "=== Node Taints ==="
kubectl describe nodes -l agentpool=$NODEPOOL_NAME | grep -A 5 Taints

# Get specific nodes in the pool
BATCH_NODES=$(kubectl get nodes -l agentpool=$NODEPOOL_NAME \
  --output jsonpath='{.items[*].metadata.name}')
echo ""
echo "Batch pool nodes: $BATCH_NODES"
```

---

## 5. Deploy Workload WITHOUT Tolerations (Expected: Fails)
---

## 5. Deploy Workload WITHOUT Tolerations (Expected: Fails)

Create `batch-job-no-toleration.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-fail
spec:
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      nodeSelector:
        workload: batch      # Tries to select batch nodes
      containers:
      - name: processor
        image: busybox
        command: ["sh", "-c", "echo Processing && sleep 30"]
      restartPolicy: Never
  backoffLimit: 2
```

Deploy and check:
```bash
# Apply the job
JOB_RESULT=$(kubectl apply -f batch-job-no-toleration.yaml)
echo "$JOB_RESULT"

# Wait a few seconds for scheduling attempt
sleep 5

# Check pod status (should be Pending)
echo ""
echo "=== Pod Status ==="
kubectl get pods -l app=batch-processor

# Get pod name
FAILED_POD=$(kubectl get pods -l app=batch-processor \
  --output jsonpath='{.items[0].metadata.name}')

# Describe pod to see why it's pending
echo ""
echo "=== Pod Events (check for taint-related errors) ==="
kubectl describe pod $FAILED_POD | grep -A 10 Events
```

**Expected:** Pod stays in `Pending` state because it has `nodeSelector` for batch nodes but lacks the required toleration for the `workload=batch:NoSchedule` taint.

---

## 6. Deploy Workload WITH Tolerations (Expected: Succeeds)
---

## 6. Deploy Workload WITH Tolerations (Expected: Succeeds)

Create `batch-job-with-toleration.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-success
spec:
  parallelism: 3          # Run 3 pods in parallel
  completions: 3          # Total 3 completions required
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      nodeSelector:
        workload: batch   # Select batch nodes
      tolerations:
      - key: workload     # Tolerate the batch taint
        operator: Equal
        value: batch
        effect: NoSchedule
      containers:
      - name: processor
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Processing on node: $NODE_NAME"
          echo "Started at: $(date)"
          sleep 60
          echo "Completed at: $(date)"
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
      restartPolicy: Never
  backoffLimit: 3
```

Apply and monitor:
```bash
# Apply the job
JOB_SUCCESS_RESULT=$(kubectl apply -f batch-job-with-toleration.yaml)
echo "$JOB_SUCCESS_RESULT"

# Wait for pods to be scheduled
echo "Waiting for pods to be scheduled..."
sleep 10

# Check pod status and placement
echo ""
echo "=== Pod Status and Placement ==="
kubectl get pods -l app=batch-processor -o wide

# Get pod count on batch nodes
POD_COUNT=$(kubectl get pods -l app=batch-processor \
  --field-selector status.phase=Running \
  --output json | jq '.items | length')
echo ""
echo "Running pods: $POD_COUNT"

# Follow logs from one of the pods
echo ""
echo "=== Sample Pod Logs ==="
kubectl logs -l app=batch-processor --tail=10 | head -20
```

**Expected:** Pods successfully scheduled on `batchpool` nodes.

---

## 7. Deploy DaemonSet on Batch Nodes
---

## 7. Deploy DaemonSet on Batch Nodes

Create `batch-monitoring-daemonset.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: batch-node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      nodeSelector:
        workload: batch    # Only run on batch nodes
      tolerations:
      - key: workload      # Tolerate batch taint
        operator: Equal
        value: batch
        effect: NoSchedule
      containers:
      - name: monitor
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            echo "Monitoring node: $NODE_NAME at $(date)"
            sleep 60
          done
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```

Deploy and verify:
```bash
# Apply DaemonSet
DS_RESULT=$(kubectl apply -f batch-monitoring-daemonset.yaml)
echo "$DS_RESULT"

# Wait for DaemonSet to deploy
sleep 5

# Check DaemonSet status
echo ""
echo "=== DaemonSet Status ==="
kubectl get daemonset batch-node-monitor

# Verify one pod per batch node
echo ""
echo "=== DaemonSet Pods (should match number of batch nodes) ==="
kubectl get pods -l app=node-monitor -o wide

# Count DaemonSet pods
DS_POD_COUNT=$(kubectl get pods -l app=node-monitor \
  --output json | jq '.items | length')
echo ""
echo "DaemonSet pods running: $DS_POD_COUNT (should equal $NODE_COUNT)"

# Check logs from one monitor pod
echo ""
echo "=== Sample Monitor Logs ==="
kubectl logs -l app=node-monitor --tail=5 | head -10
```

---

## 8. Advanced Scheduling: Node Affinity
Create `advanced-scheduling.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-workload
spec:
  replicas: 2
  selector:
    matchLabels:
      app: batch-app
  template:
    metadata:
      labels:
        app: batch-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload
                operator: In
                values:
                - batch
      tolerations:
      - key: workload
        operator: Equal
        value: batch
        effect: NoSchedule
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

---

## 8. Advanced Scheduling: Node Affinity

Create `advanced-scheduling.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-workload
spec:
  replicas: 2
  selector:
    matchLabels:
      app: batch-app
  template:
    metadata:
      labels:
        app: batch-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload
                operator: In
                values:
                - batch
      tolerations:
      - key: workload
        operator: Equal
        value: batch
        effect: NoSchedule
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

Deploy and verify:
```bash
# Apply deployment
DEPLOY_RESULT=$(kubectl apply -f advanced-scheduling.yaml)
echo "$DEPLOY_RESULT"

# Wait for pods to be ready
echo "Waiting for deployment to be ready..."
kubectl wait --for=condition=available \
  --timeout=60s \
  deployment/batch-workload

# Check pod placement
echo ""
echo "=== Deployment Pod Placement ==="
kubectl get pods -l app=batch-app -o wide

# Verify all pods are on batch nodes
BATCH_APP_NODES=$(kubectl get pods -l app=batch-app \
  --output jsonpath='{.items[*].spec.nodeName}')
echo ""
echo "Pods running on nodes: $BATCH_APP_NODES"

# Get deployment status
READY_REPLICAS=$(kubectl get deployment batch-workload \
  --output jsonpath='{.status.readyReplicas}')
echo "Ready replicas: $READY_REPLICAS/2"
```

---

## Expected Results

✅ A dedicated user node pool is created successfully  
✅ Pods without tolerations fail to schedule on tainted nodes  
✅ Pods with tolerations schedule correctly to the batch node pool  
✅ DaemonSet runs one pod per batch node  
✅ Node affinity ensures workloads run only on batch nodes  

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete all jobs
kubectl delete job batch-job-fail batch-job-success

# Delete DaemonSet
kubectl delete daemonset batch-node-monitor

# Delete deployment
kubectl delete deployment batch-workload

# Verify cleanup
echo "Remaining workloads:"
kubectl get all -l app=batch-processor
kubectl get all -l app=node-monitor
kubectl get all -l app=batch-app
```

### Option 2: Delete Node Pool
```bash
# Delete the user node pool (keeps cluster)
echo "Deleting node pool: $NODEPOOL_NAME..."
NODEPOOL_DELETE=$(az aks nodepool delete \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NODEPOOL_NAME \
  --no-wait \
  --output tsv)

echo "Node pool deletion initiated"

# Verify remaining node pools
echo ""
echo "Remaining node pools:"
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --output table
```

### Option 3: Delete Entire Resource Group (Complete Cleanup)
```bash
# Delete entire resource group
echo "Deleting resource group: $RESOURCE_GROUP..."
DELETE_RESULT=$(az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait \
  --output tsv)

echo "Delete operation initiated: $DELETE_RESULT"

# Verify deletion status
echo ""
echo "To verify deletion status, run:"
echo "az group show --name $RESOURCE_GROUP --query properties.provisioningState"
```

---

## Key Takeaways

- **Labels and taints** define where workloads should run
- **Tolerations** allow workloads to run on otherwise restricted nodes
- **Node pools** isolate compute for special workloads (batch, GPU, ML)
- **Node affinity** gives fine-grained control over pod placement
- **AKS autoscaler** works per node pool, enabling independent scaling

---

## Troubleshooting

- **Pods stuck in Pending** → missing toleration or insufficient nodes
- **Node pool missing** → check with `az aks nodepool list`
- **Autoscaler not scaling** → verify min/max and resource requests
- **DaemonSet pods not matching node count** → check tolerations and nodeSelector

---



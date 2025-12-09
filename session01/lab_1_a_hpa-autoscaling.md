# Lab 1a: HPA Autoscaling

## Objective
Deploy a CPU-intensive workload and observe HPA-driven pod autoscaling.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Lab configuration variables
RESOURCE_GROUP="rg-aks-hpa-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-hpa-demo"
NODE_COUNT=2
NODE_VM_SIZE="Standard_D2s_v3"
MIN_NODE_COUNT=2
MAX_NODE_COUNT=5

# Display configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Node Count: $NODE_COUNT"
echo "Node VM Size: $NODE_VM_SIZE"
echo "Autoscaler Min: $MIN_NODE_COUNT"
echo "Autoscaler Max: $MAX_NODE_COUNT"
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

# Create AKS cluster with metrics server enabled
echo "Creating AKS cluster (this may take 5-10 minutes)..."
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $NODE_VM_SIZE \
  --enable-managed-identity \
  --enable-cluster-autoscaler \
  --min-count $MIN_NODE_COUNT \
  --max-count $MAX_NODE_COUNT \
  --generate-ssh-keys \
  --query 'provisioningState' \
  --output tsv)

echo "AKS cluster creation: $CLUSTER_RESULT"

# Get AKS credentials and store cluster FQDN
CLUSTER_FQDN=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query 'fqdn' \
  --output tsv)

echo "Cluster FQDN: $CLUSTER_FQDN"

# Configure kubectl context
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

## 3. Verify Metrics Server

```bash
# Check if metrics server is running
METRICS_SERVER_STATUS=$(kubectl get deployment metrics-server \
  -n kube-system \
  --ignore-not-found=true \
  --output jsonpath='{.status.availableReplicas}' 2>/dev/null)

if [ -z "$METRICS_SERVER_STATUS" ] || [ "$METRICS_SERVER_STATUS" == "0" ]; then
  echo "Metrics server not available. Enabling..."
  
  # Enable metrics server on AKS
  az aks update \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --enable-metrics-server
  
  echo "Waiting for metrics server to be ready..."
  sleep 30
else
  echo "Metrics server is running with $METRICS_SERVER_STATUS replicas"
fi

# Verify metrics server is responding
echo "Testing metrics server..."
kubectl top nodes || echo "Metrics not yet available (may take 1-2 minutes after enabling)"
```

---

## 4. Deploy CPU-Intensive Application

```bash
# Application configuration variables
APP_NAME="cpu-stress"
APP_NAMESPACE="default"
CPU_REQUEST="100m"
CPU_LIMIT="200m"
MEMORY_REQUEST="128Mi"
MEMORY_LIMIT="256Mi"
REPLICAS=1

echo "=== Application Configuration ==="
echo "App Name: $APP_NAME"
echo "Namespace: $APP_NAMESPACE"
echo "CPU Request: $CPU_REQUEST"
echo "CPU Limit: $CPU_LIMIT"
echo "Memory Request: $MEMORY_REQUEST"
echo "Memory Limit: $MEMORY_LIMIT"
echo "Initial Replicas: $REPLICAS"
echo "================================="
```

Create `cpu-stress-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-stress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-stress
  template:
    metadata:
      labels:
        app: cpu-stress
    spec:
      containers:
      - name: cpu-stress
        image: containerstack/cpustress
        resources:
          requests:
            cpu: 100m      # Minimum CPU required for scheduling
            memory: 128Mi  # Minimum memory required
          limits:
            cpu: 200m      # Maximum CPU the container can use
            memory: 256Mi  # Maximum memory the container can use
        command: ["/app/cpustress"]
        args: ["-cpus", "1"]  # Stress 1 CPU core
```

Apply the deployment:
```bash
# Create the deployment
DEPLOY_RESULT=$(kubectl apply -f cpu-stress-deployment.yaml)
echo "$DEPLOY_RESULT"

# Wait for deployment to be ready
echo "Waiting for deployment to be ready..."
kubectl wait --for=condition=available \
  --timeout=120s \
  deployment/$APP_NAME

# Verify deployment is running
DEPLOYMENT_STATUS=$(kubectl get deployment $APP_NAME \
  --output jsonpath='{.status.availableReplicas}')
echo "Deployment $APP_NAME has $DEPLOYMENT_STATUS available replicas"

# Get pod details
echo "Pod status:"
POD_NAME=$(kubectl get pods -l app=$APP_NAME \
  --output jsonpath='{.items[0].metadata.name}')
echo "Pod name: $POD_NAME"

kubectl get pods -l app=$APP_NAME
```

---

## 5. Create Horizontal Pod Autoscaler (HPA)

```bash
# HPA configuration variables
HPA_NAME="cpu-stress-hpa"
HPA_MIN_REPLICAS=1
HPA_MAX_REPLICAS=10
HPA_CPU_TARGET=50

echo "=== HPA Configuration ==="
echo "HPA Name: $HPA_NAME"
echo "Min Replicas: $HPA_MIN_REPLICAS"
echo "Max Replicas: $HPA_MAX_REPLICAS"
echo "CPU Target: ${HPA_CPU_TARGET}%"
echo "========================="
```

**Option A - Imperative:**
```bash
# Create HPA using kubectl command
HPA_CREATE_RESULT=$(kubectl autoscale deployment $APP_NAME \
  --name=$HPA_NAME \
  --cpu-percent=$HPA_CPU_TARGET \
  --min=$HPA_MIN_REPLICAS \
  --max=$HPA_MAX_REPLICAS)

echo "$HPA_CREATE_RESULT"
```

**Option B - Declarative (Recommended):**

Create `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-stress-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-stress              # Target deployment to scale
  minReplicas: 1                  # Minimum number of replicas
  maxReplicas: 10                 # Maximum number of replicas
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50    # Scale when average CPU exceeds 50%
```

Apply the HPA:
```bash
# Create HPA resource
HPA_APPLY_RESULT=$(kubectl apply -f hpa.yaml)
echo "$HPA_APPLY_RESULT"

# Wait a few seconds for HPA to initialize
sleep 5

# Verify HPA is created and get current status
HPA_STATUS=$(kubectl get hpa $HPA_NAME \
  --output jsonpath='{.status.conditions[?(@.type=="ScalingActive")].status}')
echo "HPA ScalingActive status: $HPA_STATUS"

# Display HPA details
kubectl get hpa $HPA_NAME

# View detailed HPA information
echo ""
echo "=== HPA Details ==="
kubectl describe hpa $HPA_NAME
```

---

## 6. Generate Load

**Option A - Increase CPU stress directly (Simple):**
```bash
# Increase the number of replicas to trigger higher CPU usage
# This will cause existing pods to consume more CPU
echo "Scaling deployment to trigger CPU load..."
kubectl scale deployment $APP_NAME --replicas=3

# Wait for pods to start
sleep 10

# Verify all pods are consuming CPU
kubectl top pods -l app=$APP_NAME
```

**Option B - Run additional CPU stress pod (Recommended):**
```bash
# Create a second CPU-intensive deployment to increase cluster load
LOAD_GEN_NAME="cpu-stress-load"

echo "Creating additional CPU load generator: $LOAD_GEN_NAME"

# Deploy another CPU stress instance
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $LOAD_GEN_NAME
spec:
  replicas: 2
  selector:
    matchLabels:
      app: $LOAD_GEN_NAME
  template:
    metadata:
      labels:
        app: $LOAD_GEN_NAME
    spec:
      containers:
      - name: stress
        image: containerstack/cpustress
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m      # Higher limit to stress more
            memory: 256Mi
        command: ["/app/cpustress"]
        args: ["-cpus", "2"]  # Stress 2 CPU cores
EOF

# Wait for load generator to start
echo "Waiting for load generator pods to start..."
sleep 15

# Verify load generator is running
LOAD_GEN_STATUS=$(kubectl get deployment $LOAD_GEN_NAME \
  --output jsonpath='{.status.readyReplicas}')
echo "Load generator ready replicas: $LOAD_GEN_STATUS"

# Check combined CPU usage
echo ""
echo "Combined CPU usage:"
kubectl top pods
```

**Option C - Exec into pod and increase stress (Interactive):**
```bash
# Get the pod name
POD_NAME=$(kubectl get pods -l app=$APP_NAME \
  --output jsonpath='{.items[0].metadata.name}')

echo "Current pod: $POD_NAME"

# Note: The cpu-stress container will automatically consume CPU
# To verify it's working, check metrics:
kubectl top pod $POD_NAME
```

---

## 7. Monitor HPA Behavior

```bash
# Get initial replica count
INITIAL_REPLICAS=$(kubectl get deployment $APP_NAME \
  --output jsonpath='{.spec.replicas}')
echo "Initial replicas: $INITIAL_REPLICAS"

# Watch HPA status in real-time (shows current/target CPU and replica count)
echo ""
echo "=== Monitoring HPA (Ctrl+C to stop) ==="
echo "Watch for CPU% to exceed target and replicas to increase..."
kubectl get hpa $HPA_NAME --watch
```

In a separate terminal, run these monitoring commands:
```bash
# Set variables (if in new terminal)
APP_NAME="cpu-stress"
HPA_NAME="cpu-stress-hpa"

# Watch pods being created/terminated
kubectl get pods -l app=$APP_NAME --watch
```

Additional monitoring commands:
```bash
# View detailed HPA events and conditions
echo "=== HPA Detailed Status ==="
kubectl describe hpa $HPA_NAME

# Check current CPU/memory usage per pod
echo ""
echo "=== Pod Resource Usage ==="
kubectl top pods -l app=$APP_NAME

# Get current replica count
CURRENT_REPLICAS=$(kubectl get deployment $APP_NAME \
  --output jsonpath='{.status.replicas}')
READY_REPLICAS=$(kubectl get deployment $APP_NAME \
  --output jsonpath='{.status.readyReplicas}')
echo ""
echo "Current replicas: $CURRENT_REPLICAS"
echo "Ready replicas: $READY_REPLICAS"

# Check node resource utilization
echo ""
echo "=== Node Resource Usage ==="
kubectl top nodes

# View HPA scaling events
echo ""
echo "=== Recent HPA Events ==="
kubectl get events \
  --field-selector involvedObject.name=$HPA_NAME \
  --sort-by='.lastTimestamp' \
  | tail -20

# Monitor scaling metrics in real-time
echo ""
echo "=== Current HPA Metrics ==="
HPA_CURRENT_CPU=$(kubectl get hpa $HPA_NAME \
  --output jsonpath='{.status.currentMetrics[0].resource.current.averageUtilization}')
HPA_TARGET_CPU=$(kubectl get hpa $HPA_NAME \
  --output jsonpath='{.spec.metrics[0].resource.target.averageUtilization}')
HPA_CURRENT_REPLICAS=$(kubectl get hpa $HPA_NAME \
  --output jsonpath='{.status.currentReplicas}')
HPA_DESIRED_REPLICAS=$(kubectl get hpa $HPA_NAME \
  --output jsonpath='{.status.desiredReplicas}')

echo "Current CPU: ${HPA_CURRENT_CPU}%"
echo "Target CPU: ${HPA_TARGET_CPU}%"
echo "Current Replicas: $HPA_CURRENT_REPLICAS"
echo "Desired Replicas: $HPA_DESIRED_REPLICAS"
```

---

## 🔍 Understanding HPA v2 Behavior

### Stabilization Windows
HPA v2 uses stabilization windows to prevent "flapping" (rapid scaling up and down):

- **Scale-up window**: Default 0 seconds (scales up immediately when threshold exceeded)
- **Scale-down window**: Default 300 seconds (5 minutes - waits before scaling down)

This asymmetric behavior ensures:
- Quick response to increased load
- Stability during temporary CPU spikes
- Gradual scale-down to avoid service disruption

---

## Expected Results
- HPA scales pods from **1 → multiple replicas** when CPU exceeds 50%
- Scaling occurs within **1–3 minutes**
- Scale-down occurs after the stabilization window (~5 minutes)

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Set variables if needed
APP_NAME="cpu-stress"
HPA_NAME="cpu-stress-hpa"
LOAD_GEN_NAME="load-generator"

# Delete HPA
kubectl delete hpa $HPA_NAME

# Delete deployment
kubectl delete deployment $APP_NAME

# Delete service (if any)
kubectl delete service $APP_NAME --ignore-not-found=true

# Delete load generator deployment (if created)
kubectl delete deployment cpu-stress-load --ignore-not-found=true

# Verify cleanup
echo "Remaining resources:"
kubectl get all -l app=$APP_NAME
```

### Option 2: Delete Entire Resource Group (Complete Cleanup)
```bash
# Set variables if needed
RESOURCE_GROUP="rg-aks-hpa-lab"

# Delete the entire resource group (removes AKS cluster and all resources)
echo "Deleting resource group $RESOURCE_GROUP..."
DELETE_RESULT=$(az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait \
  --output tsv)

echo "Delete operation initiated: $DELETE_RESULT"

# Verify deletion (optional - check after a few minutes)
echo ""
echo "To verify deletion status, run:"
echo "az group show --name $RESOURCE_GROUP --query properties.provisioningState"
```

---

## 📘 How This Lab Connects to Cluster Autoscaler (Next Lab)

This lab introduces **pod-level scaling** via HPA.  
Later labs will demonstrate how HPA and Cluster Autoscaler work together:

- When HPA increases replicas, node capacity may become insufficient  
- Cluster Autoscaler (CA) automatically **adds new nodes** to accommodate pod scheduling  
- Together, HPA (pods) + CA (nodes) enable **full-stack autoscaling** in AKS  

Mastering HPA here is essential before learning **node-level autoscaling**.

---

## Key Takeaways
- HPA scales pods automatically based on CPU demand  
- Stabilization windows prevent unstable oscillations  
- Resource requests are required for accurate HPA decisions  
- HPA behavior directly influences Cluster Autoscaler actions  

---

## Troubleshooting
- `<unknown>` metrics → Metrics Server not running  
- No scaling → CPU threshold not exceeded  
- Slow scaling → stabilization window delays decisions  

---



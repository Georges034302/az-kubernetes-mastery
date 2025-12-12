# Lab 4a: KQL Pod Restarts Analysis

## Objective
Analyze pod restarts, failures, and resource issues in AKS using **Azure Monitor Container Insights** and **Kusto Query Language (KQL)** to identify problematic workloads and root causes.

---

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Subscription with permissions to create resources
- Basic understanding of KQL syntax

---

## 1. Define Environment Variables

```bash
# Resource naming
RESOURCE_GROUP="rg-aks-kql-analysis"
LOCATION="australiaeast"
CLUSTER_NAME="aks-kql-analysis"
WORKSPACE_NAME="log-aks-monitoring"
TEST_NAMESPACE="restart-test"

# AKS configuration
NODE_COUNT=2
NODE_SIZE="Standard_DS2_v2"
K8S_VERSION="1.29"
```

Display variables:
```bash
echo "RESOURCE_GROUP=$RESOURCE_GROUP"
echo "LOCATION=$LOCATION"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "WORKSPACE_NAME=$WORKSPACE_NAME"
echo "TEST_NAMESPACE=$TEST_NAMESPACE"
echo "NODE_COUNT=$NODE_COUNT"
echo "NODE_SIZE=$NODE_SIZE"
echo "K8S_VERSION=$K8S_VERSION"
```

---

## 2. Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP `# Target resource group` \
  --location $LOCATION `# Azure region`
```

---

## 3. Create Log Analytics Workspace

```bash
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --workspace-name $WORKSPACE_NAME `# Workspace name` \
  --location $LOCATION `# Azure region`
```

Retrieve workspace resource ID:
```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME \
  --query id -o tsv)

echo "WORKSPACE_ID=$WORKSPACE_ID"
```

---

## 4. Create AKS Cluster with Container Insights

```bash
az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --kubernetes-version $K8S_VERSION `# Kubernetes version` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_SIZE `# Node VM size` \
  --enable-addons monitoring `# Enable Container Insights addon` \
  --workspace-resource-id $WORKSPACE_ID `# Log Analytics workspace ID` \
  --generate-ssh-keys `# Generate SSH keys automatically` \
  --enable-managed-identity `# Use managed identity` \
  --no-wait `# Run asynchronously`
```

Wait for cluster creation:
```bash
az aks wait \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --created \
  --interval 30 \
  --timeout 1800
```

Get cluster credentials:
```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --overwrite-existing `# Overwrite existing context`
```

---

## 5. Create Test Namespace

```bash
kubectl create namespace $TEST_NAMESPACE
```

---

## 6. Deploy Applications with Restart Scenarios

### Deploy OOMKilled Application (Memory Limit Exceeded)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-leak-app
  namespace: $TEST_NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: memory-leak
  template:
    metadata:
      labels:
        app: memory-leak
    spec:
      containers:
      - name: leaker
        image: polinux/stress
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "100Mi"
          limits:
            memory: "200Mi"
EOF
```

### Deploy CrashLoopBackOff Application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-loop-app
  namespace: $TEST_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-loop
  template:
    metadata:
      labels:
        app: crash-loop
    spec:
      containers:
      - name: crasher
        image: busybox
        command: ["sh", "-c", "echo 'Starting application...'; sleep 5; exit 1"]
EOF
```

### Deploy ImagePullBackOff Application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: imagepull-fail-app
  namespace: $TEST_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: imagepull-fail
  template:
    metadata:
      labels:
        app: imagepull-fail
    spec:
      containers:
      - name: badimage
        image: nonexistent/fake-repository:v1.0
EOF
```

**Note**: Wait 5-10 minutes for restart events to accumulate and be ingested into Log Analytics.

---

## 7. Query Pod Restarts with KQL

### Query 1: Pod Restart Counts (Last 24 Hours)

**KQL Query**:
```kql
KubePodInventory
| where TimeGenerated > ago(24h)
| where RestartCount > 0
| summarize TotalRestarts = sum(RestartCount) by Namespace, ControllerName, ContainerName
| order by TotalRestarts desc
```

**Run via CLI**:
```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID `# Log Analytics workspace ID` \
  --analytics-query "
    KubePodInventory
    | where TimeGenerated > ago(24h)
    | where RestartCount > 0
    | summarize TotalRestarts = sum(RestartCount) by Namespace, ControllerName, ContainerName
    | order by TotalRestarts desc
  " `# KQL query for restart counts` \
  -o table `# Output as table format`
```

### Query 2: Identify OOMKilled Containers

**KQL Query**:
```kql
ContainerInventory
| where TimeGenerated > ago(6h)
| where ContainerState == "OOMKilled" or ContainerStatusReason == "OOMKilled"
| project TimeGenerated, Computer, Image, Name, ContainerState, ContainerStatusReason
| order by TimeGenerated desc
```

### Query 3: Pod Restart Trend Over Time

**KQL Query**:
```kql
KubePodInventory
| where TimeGenerated > ago(24h)
| summarize RestartCount = sum(RestartCount) by bin(TimeGenerated, 1h), Namespace
| render timechart
```

---

## 8. Query Failed Deployments

### Query 4: Failed Pod Starts

**KQL Query**:
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Reason in ('Failed', 'FailedScheduling', 'FailedMount')
| project TimeGenerated, Namespace, Name, Reason, Message
| order by TimeGenerated desc
```

**Run via CLI**:
```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID `# Log Analytics workspace ID` \
  --analytics-query "
    KubeEvents
    | where TimeGenerated > ago(6h)
    | where Reason in ('Failed', 'FailedScheduling', 'FailedMount')
    | project TimeGenerated, Namespace, Name, Reason, Message
    | order by TimeGenerated desc
  " `# KQL query for failed events` \
  -o table `# Output as table format`
```

### Query 5: ImagePullBackOff Errors

**KQL Query**:
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Message contains "ImagePullBackOff" or Message contains "ErrImagePull"
| project TimeGenerated, Namespace, Name, Message
| order by TimeGenerated desc
```

### Query 6: CrashLoopBackOff Events

**KQL Query**:
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Reason == "BackOff"
| project TimeGenerated, Namespace, Name, Message
| order by TimeGenerated desc
```

---

## 9. Query Container Logs for Errors

### Query 7: Container Logs with Errors

**KQL Query**:
```kql
ContainerLog
| where TimeGenerated > ago(1h)
| where LogEntry contains "error" or LogEntry contains "ERROR" or LogEntry contains "exception"
| project TimeGenerated, Computer, ContainerID, Image, LogEntry
| order by TimeGenerated desc
| take 100
```

### Query 8: Specific Container Logs (CrashLoop App)

**KQL Query**:
```kql
ContainerLog
| where TimeGenerated > ago(6h)
| where Name contains "crash-loop"
| project TimeGenerated, LogEntry
| order by TimeGenerated desc
```

---

## 10. Query Deployment and Pod Health

### Query 9: Pods Not in Running State

**KQL Query**:
```kql
KubePodInventory
| where TimeGenerated > ago(30m)
| where PodStatus != "Running" and PodStatus != "Succeeded"
| summarize arg_max(TimeGenerated, *) by Name
| project TimeGenerated, Namespace, Name, PodStatus, ContainerStatusReason
| order by TimeGenerated desc
```

### Query 10: Pod Status Summary

**KQL Query**:
```kql
KubePodInventory
| where TimeGenerated > ago(30m)
| summarize arg_max(TimeGenerated, *) by Name
| summarize Count = count() by PodStatus, Namespace
| order by Namespace, Count desc
```

---

## 11. Query Resource Utilization Issues

### Query 11: High Memory Usage Pods

**KQL Query**:
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "K8SContainer"
| where CounterName == "memoryRssBytes"
| summarize AvgMemory = avg(CounterValue) by Computer, InstanceName
| where AvgMemory > 200000000
| order by AvgMemory desc
```

### Query 12: CPU Throttling Detection

**KQL Query**:
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "K8SContainer"
| where CounterName == "cpuUsageNanoCores"
| summarize MaxCPU = max(CounterValue), AvgCPU = avg(CounterValue) by bin(TimeGenerated, 5m), InstanceName
| order by TimeGenerated desc
```

---

## 12. Advanced Root Cause Analysis

### Query 13: Correlation Between Restarts and Resource Limits

**KQL Query**:
```kql
let RestartingPods = KubePodInventory
| where TimeGenerated > ago(6h)
| where RestartCount > 0
| distinct Name;
Perf
| where TimeGenerated > ago(6h)
| where ObjectName == "K8SContainer"
| where InstanceName in (RestartingPods)
| where CounterName in ("memoryRssBytes", "cpuUsageNanoCores")
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), InstanceName, CounterName
| render timechart
```

### Query 14: Pod Lifecycle Timeline

**KQL Query**:
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Name contains "memory-leak"
| project TimeGenerated, Type, Reason, Message
| order by TimeGenerated asc
```

---

## 13. Query Node-Level Issues

### Query 15: Node Conditions

**KQL Query**:
```kql
KubeNodeInventory
| where TimeGenerated > ago(1h)
| summarize arg_max(TimeGenerated, *) by Computer
| project Computer, Status, KubeletVersion, MemoryPressure, DiskPressure
```

### Query 16: Evicted Pods

**KQL Query**:
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Reason == "Evicted"
| project TimeGenerated, Namespace, Name, Message
| order by TimeGenerated desc
```

---

## 14. Create Alert for High Restart Count

```bash
az monitor scheduled-query create \
  --name High-Pod-Restart-Alert `# Alert rule name` \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --scopes $WORKSPACE_ID `# Log Analytics workspace ID` \
  --condition "count 'KubePodInventory' > 0" `# Condition expression` \
  --condition-query "
    KubePodInventory
    | where RestartCount > 5
    | summarize AggregatedValue = count() by bin(TimeGenerated, 5m)
  " `# KQL query for alert condition` \
  --description "Alert when pods restart more than 5 times" `# Alert description` \
  --evaluation-frequency 5m `# How often to run query` \
  --window-size 15m `# Time window for evaluation` \
  --severity 2 `# Alert severity (0-4)`
```

---

## 15. Query Overall Cluster Health

### Query 17: Overall Cluster Health Summary

**KQL Query**:
```kql
KubePodInventory
| where TimeGenerated > ago(30m)
| summarize arg_max(TimeGenerated, *) by Name
| summarize 
    TotalPods = count(),
    RunningPods = countif(PodStatus == "Running"),
    FailedPods = countif(PodStatus == "Failed"),
    PendingPods = countif(PodStatus == "Pending"),
    TotalRestarts = sum(RestartCount)
| extend HealthPercentage = round(RunningPods * 100.0 / TotalPods, 2)
```

---

## 16. Cleanup

**Option 1**: Remove test resources only:
```bash
kubectl delete namespace $TEST_NAMESPACE
```

**Option 2**: Delete entire resource group:
```bash
az group delete \
  --name $RESOURCE_GROUP `# Resource group to delete` \
  --yes `# Skip confirmation` \
  --no-wait `# Run asynchronously`
```

---

## How This Connects to AKS Observability

### Container Insights Architecture
```
        ┌──────────────────────────────┐
        │   AKS Cluster                │
        │  ┌────────────────────────┐  │
        │  │  OMS Agent (DaemonSet) │  │
        │  │  • Collects metrics    │  │
        │  │  • Collects logs       │  │
        │  │  • Scrapes kubelet     │  │
        │  └─────────┬──────────────┘  │
        └────────────┼──────────────────┘
                     │ HTTPS
        ┌────────────▼──────────────────┐
        │  Log Analytics Workspace      │
        │  • KubePodInventory          │
        │  • KubeEvents                │
        │  • ContainerLog              │
        │  • Perf                      │
        │  • ContainerInventory        │
        └────────────┬──────────────────┘
                     │
        ┌────────────▼──────────────────┐
        │  Azure Monitor                │
        │  • KQL Queries               │
        │  • Alerts                    │
        │  • Workbooks                 │
        │  • Dashboards                │
        └──────────────────────────────┘
```

### Key Container Insights Tables

| Table | Purpose | Key Fields |
|-------|---------|------------|
| **KubePodInventory** | Pod state and metadata | PodStatus, RestartCount, Namespace, ControllerName |
| **KubeEvents** | Kubernetes events | Reason, Message, Type (Warning/Normal) |
| **ContainerLog** | Container stdout/stderr | LogEntry, Name, Image |
| **Perf** | Resource metrics | CounterName, CounterValue (CPU, memory) |
| **ContainerInventory** | Container state | ContainerState, ContainerStatusReason |
| **KubeNodeInventory** | Node health | Status, MemoryPressure, DiskPressure |

### Common KQL Operators and Functions

| Operator/Function | Purpose | Example |
|-------------------|---------|---------|
| `ago()` | Relative time filter | `ago(1h)`, `ago(24h)`, `ago(7d)` |
| `bin()` | Time bucketing for aggregation | `bin(TimeGenerated, 5m)` |
| `summarize` | Aggregate data | `summarize count() by Namespace` |
| `where` | Filter rows | `where RestartCount > 0` |
| `project` | Select specific columns | `project TimeGenerated, Name, Status` |
| `order by` | Sort results | `order by TimeGenerated desc` |
| `join` | Combine tables | `join kind=inner (Table2) on Key` |
| `render` | Visualize data | `render timechart`, `render barchart` |
| `arg_max()` | Get latest record per group | `arg_max(TimeGenerated, *)` |
| `take` | Limit results | `take 100` |

### Pod Restart Root Causes

| Failure Type | Symptom | KQL Detection | Root Cause |
|--------------|---------|---------------|------------|
| **OOMKilled** | Pod restarts, memory limit exceeded | `ContainerStatusReason == "OOMKilled"` | Memory limit too low or memory leak |
| **CrashLoopBackOff** | Repeated container crashes | `Reason == "BackOff"` in KubeEvents | Application error, missing dependencies |
| **ImagePullBackOff** | Cannot pull container image | `Message contains "ImagePullBackOff"` | Invalid image name, auth issues, registry down |
| **Liveness Probe Failure** | Health check failures | `Reason == "Unhealthy"` in KubeEvents | App unresponsive, misconfigured probe |
| **Node Pressure Eviction** | Pod evicted from node | `Reason == "Evicted"` in KubeEvents | Node resource exhaustion |

### KQL Query Patterns

**Time-series aggregation**:
```kql
KubePodInventory
| summarize count() by bin(TimeGenerated, 1h)
| render timechart
```

**Top N analysis**:
```kql
KubePodInventory
| summarize TotalRestarts = sum(RestartCount) by ControllerName
| top 10 by TotalRestarts desc
```

**Correlation with let statement**:
```kql
let FailedPods = KubePodInventory | where PodStatus == "Failed" | distinct Name;
KubeEvents | where Name in (FailedPods)
```

**Join across tables**:
```kql
KubePodInventory
| join kind=inner (KubeEvents) on Name
| where Reason == "Failed"
```

---

## Expected Results

✅ **AKS cluster deployed** with Container Insights enabled  
✅ **Log Analytics workspace** collecting pod, event, and performance data  
✅ **Test applications deployed** exhibiting restart scenarios (OOMKilled, CrashLoop, ImagePull)  
✅ **KQL queries executed** successfully via Azure Portal and CLI  
✅ **Pod restart counts** identified and aggregated by namespace/controller  
✅ **Root cause analysis** performed using resource metrics and event correlation  
✅ **Alert created** for high pod restart thresholds  
✅ **Cluster health metrics** calculated (running/failed/pending pod counts)  

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| **No data in tables** | Ingestion delay | Wait 5-10 minutes after enabling Container Insights |
| **Query timeout** | Time range too large | Narrow time window with `ago(1h)` instead of `ago(7d)` |
| **Missing KubePodInventory** | Container Insights not enabled | Verify addon with `az aks show --query addonProfiles.omsagent` |
| **Incorrect restart counts** | Duplicate rows | Use `arg_max(TimeGenerated, *) by Name` to deduplicate |
| **Empty ContainerLog** | Logs not being collected | Check OMS agent pods in kube-system namespace |
| **Alert not firing** | Wrong query syntax | Test query in Log Analytics before creating alert |

---

## Key Takeaways

- **Container Insights** provides comprehensive AKS observability through Log Analytics workspace
- **KubePodInventory** is primary table for pod state, restarts, and metadata
- **KubeEvents** captures Kubernetes lifecycle events (scheduling, failures, warnings)
- **ContainerLog** stores application logs for troubleshooting
- **Perf** table contains CPU and memory metrics for resource analysis
- **KQL** (Kusto Query Language) enables powerful data exploration and correlation
- **Time functions** like `ago()` and `bin()` are essential for temporal analysis
- **Aggregation** with `summarize` groups and calculates metrics
- **Alerts** can be created from any KQL query for proactive monitoring
- **Root cause analysis** requires correlating events, metrics, and logs across tables

---


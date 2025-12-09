# Lab 4a: KQL Pod Restarts

## Objective
Query restarts and failed deployments with KQL.

## Prerequisites
- AKS cluster with Azure Monitor Container Insights enabled
- Log Analytics workspace configured
- `kubectl` and Azure CLI configured
- Understanding of Kusto Query Language (KQL) basics

## Steps

### 1. Enable Container Insights on AKS
```bash
# Enable Container Insights addon
az aks enable-addons \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --addons monitoring

# Verify Log Analytics workspace
az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query "addonProfiles.omsagent.config.logAnalyticsWorkspaceResourceID" -o tsv
```

Get workspace details:
```bash
WORKSPACE_ID=$(az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query "addonProfiles.omsagent.config.logAnalyticsWorkspaceResourceID" -o tsv)

WORKSPACE_NAME=$(az monitor log-analytics workspace show \
  --ids $WORKSPACE_ID \
  --query name -o tsv)

RESOURCE_GROUP_WORKSPACE=$(az monitor log-analytics workspace show \
  --ids $WORKSPACE_ID \
  --query resourceGroup -o tsv)

echo "Workspace: $WORKSPACE_NAME"
echo "Resource Group: $RESOURCE_GROUP_WORKSPACE"
```

### 2. Deploy Test Applications with Restart Scenarios
Create apps that will restart for testing:

**App with OOMKilled scenario:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-leak-app
  namespace: default
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
      - name: app
        image: polinux/stress
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "100Mi"
          limits:
            memory: "200Mi"
```

**App with CrashLoopBackOff:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-loop-app
  namespace: default
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
      - name: app
        image: busybox
        command: ["sh", "-c", "echo Starting...; sleep 5; exit 1"]
```

**App with image pull failure:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: imagepull-fail-app
  namespace: default
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
      - name: app
        image: nonexistent/fake-image:v1.0
```

Apply all:
```bash
kubectl apply -f memory-leak-app.yaml
kubectl apply -f crash-loop-app.yaml
kubectl apply -f imagepull-fail-app.yaml
```

Wait a few minutes for restarts to occur:
```bash
kubectl get pods -w
```

### 3. Query Pod Restarts with KQL
Access Azure Portal > Log Analytics workspace > Logs

**Query 1: Count pod restarts by container in last 24 hours**
```kql
KubePodInventory
| where TimeGenerated > ago(24h)
| where RestartCount > 0
| summarize TotalRestarts = sum(RestartCount) by 
    Namespace, 
    ControllerName, 
    ContainerName,
    Computer
| order by TotalRestarts desc
```

Run via CLI:
```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "
    KubePodInventory
    | where TimeGenerated > ago(24h)
    | where RestartCount > 0
    | summarize TotalRestarts = sum(RestartCount) by Namespace, ControllerName, ContainerName
    | order by TotalRestarts desc
  " \
  -o table
```

**Query 2: Identify pods with OOMKilled status**
```kql
ContainerInventory
| where TimeGenerated > ago(6h)
| where ContainerState == "OOMKilled" or ContainerStatusReason == "OOMKilled"
| project TimeGenerated, Computer, Image, Name, ContainerState, ContainerStatusReason
| order by TimeGenerated desc
```

**Query 3: Pod restart count trend over time**
```kql
KubePodInventory
| where TimeGenerated > ago(24h)
| summarize RestartCount = sum(RestartCount) by bin(TimeGenerated, 1h), Namespace
| render timechart
```

### 4. Query Failed Deployments
**Query 4: Failed pod starts**
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Reason == "Failed" or Reason == "FailedScheduling" or Reason == "FailedMount"
| project TimeGenerated, Namespace, Name, Reason, Message
| order by TimeGenerated desc
```

Run via CLI:
```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "
    KubeEvents
    | where TimeGenerated > ago(6h)
    | where Reason == 'Failed' or Reason == 'FailedScheduling'
    | project TimeGenerated, Namespace, Name, Reason, Message
    | order by TimeGenerated desc
  " \
  -o table
```

**Query 5: ImagePullBackOff errors**
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Reason == "Failed" and Message contains "ImagePullBackOff" or Message contains "ErrImagePull"
| project TimeGenerated, Namespace, Name, Message
| order by TimeGenerated desc
```

**Query 6: CrashLoopBackOff events**
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Reason == "BackOff"
| project TimeGenerated, Namespace, Name, Message
| order by TimeGenerated desc
```

### 5. Analyze Container Logs for Errors
**Query 7: Container logs with errors**
```kql
ContainerLog
| where TimeGenerated > ago(1h)
| where LogEntry contains "error" or LogEntry contains "ERROR" or LogEntry contains "exception"
| project TimeGenerated, Computer, ContainerID, Image, LogEntry
| order by TimeGenerated desc
| take 100
```

**Query 8: Specific container errors**
```kql
ContainerLog
| where TimeGenerated > ago(6h)
| where Name contains "crash-loop"
| project TimeGenerated, LogEntry
| order by TimeGenerated desc
```

### 6. Query Deployment Status
**Query 9: Pods not in Running state**
```kql
KubePodInventory
| where TimeGenerated > ago(30m)
| where PodStatus != "Running" and PodStatus != "Succeeded"
| summarize arg_max(TimeGenerated, *) by Name
| project TimeGenerated, Namespace, Name, PodStatus, ContainerStatusReason
| order by TimeGenerated desc
```

**Query 10: Count pods by status**
```kql
KubePodInventory
| where TimeGenerated > ago(30m)
| summarize arg_max(TimeGenerated, *) by Name
| summarize Count = count() by PodStatus, Namespace
| order by Namespace, Count desc
```

### 7. Query Resource Utilization Issues
**Query 11: High memory usage pods**
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "K8SContainer"
| where CounterName == "memoryRssBytes"
| summarize AvgMemory = avg(CounterValue) by Computer, InstanceName
| where AvgMemory > 200000000  // 200MB
| order by AvgMemory desc
```

**Query 12: CPU throttling detection**
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "K8SContainer"
| where CounterName == "cpuUsageNanoCores"
| summarize MaxCPU = max(CounterValue), AvgCPU = avg(CounterValue) by bin(TimeGenerated, 5m), InstanceName
| order by TimeGenerated desc
```

### 8. Create Advanced Queries for Root Cause Analysis
**Query 13: Correlation between restarts and resource limits**
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

**Query 14: Pod lifecycle timeline**
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Name contains "memory-leak"
| project TimeGenerated, Type, Reason, Message
| order by TimeGenerated asc
```

### 9. Query Node-Level Issues
**Query 15: Node conditions**
```kql
KubeNodeInventory
| where TimeGenerated > ago(1h)
| summarize arg_max(TimeGenerated, *) by Computer
| project Computer, Status, KubeletVersion, MemoryPressure, DiskPressure
```

**Query 16: Pods evicted due to resource pressure**
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Reason == "Evicted"
| project TimeGenerated, Namespace, Name, Message
| order by TimeGenerated desc
```

### 10. Create Saved Queries
Save commonly used queries in Log Analytics:

```bash
# Via Azure Portal:
# Log Analytics > Logs > Save > Save as query
# Name: "High Restart Count Pods"
# Category: "AKS Monitoring"
```

### 11. Create Alert Rules Based on Queries
**Alert for high restart count:**
```bash
az monitor scheduled-query create \
  --name "High-Pod-Restart-Count" \
  --resource-group $RESOURCE_GROUP_WORKSPACE \
  --scopes $WORKSPACE_ID \
  --condition "count 'Heartbeat' > 0" \
  --condition-query "
    KubePodInventory
    | where RestartCount > 5
    | summarize AggregatedValue = count() by bin(TimeGenerated, 5m)
  " \
  --description "Alert when pods restart more than 5 times" \
  --evaluation-frequency 5m \
  --window-size 15m \
  --severity 2
```

### 12. Query Performance Insights
**Query 17: Slowest pod starts**
```kql
KubeEvents
| where TimeGenerated > ago(24h)
| where Reason == "Started"
| join kind=inner (
    KubeEvents
    | where Reason == "Created"
    | project CreatedTime=TimeGenerated, Name
) on Name
| extend StartDuration = datetime_diff('second', TimeGenerated, CreatedTime)
| project Name, Namespace, StartDuration
| order by StartDuration desc
| take 20
```

**Query 18: Failed health checks**
```kql
KubeEvents
| where TimeGenerated > ago(6h)
| where Reason in ("Unhealthy", "ProbeWarning")
| project TimeGenerated, Namespace, Name, Reason, Message
| order by TimeGenerated desc
```

### 13. Export Query Results
```bash
# Export to CSV
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "
    KubePodInventory
    | where TimeGenerated > ago(24h)
    | where RestartCount > 0
    | summarize TotalRestarts = sum(RestartCount) by Namespace, ControllerName
  " \
  --output table > pod_restarts.csv
```

### 14. Create Workbook for Monitoring
Navigate to Azure Portal:
- Monitor > Workbooks > New
- Add queries from above
- Create visualizations (charts, tables)
- Save as "AKS Pod Health Dashboard"

### 15. Query Summary Metrics
**Query 19: Overall cluster health**
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

## Expected Results
- Successfully query pod restart counts and trends
- Identify OOMKilled and CrashLoopBackOff scenarios
- Track failed deployments and image pull errors
- Correlate restarts with resource utilization
- Create alerts for abnormal restart patterns
- Export data for reporting and analysis

## Cleanup
```bash
kubectl delete deployment memory-leak-app crash-loop-app imagepull-fail-app
```

## Key Takeaways
- **KubePodInventory** table tracks pod state and restart counts
- **KubeEvents** table contains deployment and scheduling events
- **ContainerLog** stores container stdout/stderr logs
- **Perf** table has resource utilization metrics
- Time-based queries use `ago()` and `bin()` functions
- `summarize` aggregates data, `render` creates visualizations
- Join multiple tables for correlation analysis
- Save frequently used queries for reuse
- Create alerts based on KQL queries

## Useful KQL Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `ago()` | Relative time | `ago(1h)`, `ago(7d)` |
| `bin()` | Time bucketing | `bin(TimeGenerated, 5m)` |
| `summarize` | Aggregation | `summarize count() by Namespace` |
| `where` | Filtering | `where RestartCount > 0` |
| `join` | Combine tables | `join kind=inner` |
| `render` | Visualization | `render timechart` |
| `arg_max()` | Latest record | `arg_max(TimeGenerated, *)` |

## KQL Query Patterns

**Count by category:**
```kql
Table | summarize count() by Category
```

**Top N results:**
```kql
Table | top 10 by Value desc
```

**Time series:**
```kql
Table | summarize count() by bin(TimeGenerated, 1h) | render timechart
```

**Joins:**
```kql
Table1 | join kind=inner (Table2) on CommonField
```

## Troubleshooting
- **No data in tables**: Wait 5-10 minutes for ingestion
- **Query timeout**: Reduce time range or add filters
- **Missing tables**: Verify Container Insights is enabled
- **Incorrect results**: Check time zone and `ago()` values

## Additional Resources
- [KQL Quick Reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- [Container Insights Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-overview)
- [Log Analytics Tutorial](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-tutorial)

---


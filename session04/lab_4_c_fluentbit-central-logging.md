# Lab 4c: Fluent Bit Central Logging

## Objective
Forward AKS logs to Log Analytics using Fluent Bit.

## Prerequisites
- AKS cluster running
- Log Analytics workspace
- `kubectl` and Azure CLI configured
- Helm 3 installed

## Steps

### 1. Get Log Analytics Workspace Credentials
```bash
# Get workspace ID and key
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group <workspace-resource-group> \
  --workspace-name <workspace-name> \
  --query customerId -o tsv)

WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group <workspace-resource-group> \
  --workspace-name <workspace-name> \
  --query primarySharedKey -o tsv)

echo "Workspace ID: $WORKSPACE_ID"
```

### 2. Create Namespace for Logging
```bash
kubectl create namespace logging
```

### 3. Create Secret for Log Analytics Credentials
```bash
kubectl create secret generic log-analytics-secret \
  --from-literal=workspace-id=$WORKSPACE_ID \
  --from-literal=workspace-key=$WORKSPACE_KEY \
  -n logging
```

### 4. Install Fluent Bit with Helm
Add Fluent Bit Helm repository:

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

Create `fluent-bit-values.yaml`:
```yaml
config:
  service: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        Parsers_File parsers.conf
        Parsers_File custom_parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020
        Health_Check On

  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        multiline.parser docker, cri
        Tag kube.*
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

    [INPUT]
        Name systemd
        Tag host.*
        Systemd_Filter _SYSTEMD_UNIT=kubelet.service
        Read_From_Tail On

  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Kube_URL https://kubernetes.default.svc:443
        Kube_CA_File /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix kube.var.log.containers.
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On
        Annotations Off
        Labels On

    [FILTER]
        Name nest
        Match kube.*
        Operation lift
        Nested_under kubernetes
        Add_prefix k8s_

    [FILTER]
        Name modify
        Match kube.*
        Add source_type kubernetes

  outputs: |
    [OUTPUT]
        Name azure
        Match kube.*
        Customer_ID ${WORKSPACE_ID}
        Shared_Key ${WORKSPACE_KEY}
        Log_Type FluentBit_CL

    [OUTPUT]
        Name azure
        Match host.*
        Customer_ID ${WORKSPACE_ID}
        Shared_Key ${WORKSPACE_KEY}
        Log_Type FluentBitHost_CL

env:
  - name: WORKSPACE_ID
    valueFrom:
      secretKeyRef:
        name: log-analytics-secret
        key: workspace-id
  - name: WORKSPACE_KEY
    valueFrom:
      secretKeyRef:
        name: log-analytics-secret
        key: workspace-key

tolerations:
  - key: node-role.kubernetes.io/master
    operator: Exists
    effect: NoSchedule
  - operator: "Exists"
    effect: "NoExecute"
  - operator: "Exists"
    effect: "NoSchedule"

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

Install Fluent Bit:
```bash
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --values fluent-bit-values.yaml
```

### 5. Verify Fluent Bit Installation
```bash
# Check DaemonSet
kubectl get daemonset fluent-bit -n logging

# Check pods (one per node)
kubectl get pods -n logging -l app.kubernetes.io/name=fluent-bit

# View logs
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=50
```

### 6. Deploy Test Applications for Logging
Create apps with different log formats:

**JSON logging app:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: json-logger
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: json-logger
  template:
    metadata:
      labels:
        app: json-logger
    spec:
      containers:
      - name: logger
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            echo "{\"timestamp\":\"$(date -Iseconds)\",\"level\":\"info\",\"message\":\"Application running\",\"request_id\":\"$RANDOM\"}"
            sleep 5
          done
```

**Plain text logger:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: text-logger
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: text-logger
  template:
    metadata:
      labels:
        app: text-logger
    spec:
      containers:
      - name: logger
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            echo "$(date '+%Y-%m-%d %H:%M:%S') [INFO] Processing request $RANDOM"
            echo "$(date '+%Y-%m-%d %H:%M:%S') [WARN] High memory usage detected"
            echo "$(date '+%Y-%m-%d %H:%M:%S') [ERROR] Connection timeout to database"
            sleep 10
          done
```

**Multi-line logging app:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multiline-logger
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multiline-logger
  template:
    metadata:
      labels:
        app: multiline-logger
    spec:
      containers:
      - name: logger
        image: python:3.9-slim
        command:
        - python
        - -c
        - |
          import time
          import traceback
          while True:
              try:
                  raise Exception("Sample error for tracing")
              except Exception:
                  print(traceback.format_exc())
              time.sleep(15)
```

Apply all:
```bash
kubectl apply -f json-logger.yaml
kubectl apply -f text-logger.yaml
kubectl apply -f multiline-logger.yaml
```

### 7. Verify Logs in Log Analytics
Wait 5-10 minutes for log ingestion, then query in Azure Portal:

**Query container logs:**
```kql
FluentBit_CL
| where TimeGenerated > ago(30m)
| project TimeGenerated, log_s, k8s_pod_name_s, k8s_namespace_name_s
| order by TimeGenerated desc
| take 100
```

**Query by namespace:**
```kql
FluentBit_CL
| where k8s_namespace_name_s == "default"
| project TimeGenerated, log_s, k8s_pod_name_s
| order by TimeGenerated desc
```

**Search for errors:**
```kql
FluentBit_CL
| where log_s contains "ERROR" or log_s contains "error"
| project TimeGenerated, log_s, k8s_pod_name_s
| order by TimeGenerated desc
```

### 8. Configure Structured Logging
Update Fluent Bit to parse JSON logs:

```yaml
config:
  filters: |
    [FILTER]
        Name parser
        Match kube.*
        Key_Name log
        Parser json
        Reserve_Data On
        Preserve_Key On
```

Update Helm release:
```bash
helm upgrade fluent-bit fluent/fluent-bit \
  --namespace logging \
  --values fluent-bit-values-updated.yaml
```

### 9. Add Custom Parsers
Create custom parser configuration:

```yaml
config:
  customParsers: |
    [PARSER]
        Name custom_timestamp
        Format regex
        Regex ^(?<time>[^ ]+) \[(?<level>[^\]]+)\] (?<message>.*)$
        Time_Key time
        Time_Format %Y-%m-%d %H:%M:%S
        Time_Keep On

    [PARSER]
        Name nginx
        Format regex
        Regex ^(?<remote>[^ ]*) - - \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

  filters: |
    [FILTER]
        Name parser
        Match kube.*
        Key_Name log
        Parser custom_timestamp
        Reserve_Data On
```

### 10. Filter Logs by Namespace
Exclude certain namespaces from logging:

```yaml
config:
  filters: |
    [FILTER]
        Name grep
        Match kube.*
        Exclude k8s_namespace_name_s kube-system
```

### 11. Add Enrichment with Metadata
Enhance logs with additional context:

```yaml
config:
  filters: |
    [FILTER]
        Name modify
        Match kube.*
        Add cluster_name my-aks-cluster
        Add environment production
        Add region eastus
```

### 12. Configure Log Buffering
Add buffering for reliability:

```yaml
config:
  outputs: |
    [OUTPUT]
        Name azure
        Match kube.*
        Customer_ID ${WORKSPACE_ID}
        Shared_Key ${WORKSPACE_KEY}
        Log_Type FluentBit_CL
        Retry_Limit 3
```

### 13. Monitor Fluent Bit Metrics
Expose Fluent Bit metrics:

```bash
kubectl port-forward -n logging svc/fluent-bit 2020:2020
```

Access metrics at: http://localhost:2020/api/v1/metrics

Sample metrics:
```bash
curl http://localhost:2020/api/v1/metrics/prometheus
```

### 14. Create Alerts Based on Logs
In Log Analytics, create alert rule:

```kql
FluentBit_CL
| where log_s contains "ERROR"
| summarize ErrorCount = count() by bin(TimeGenerated, 5m), k8s_pod_name_s
| where ErrorCount > 10
```

Via CLI:
```bash
az monitor scheduled-query create \
  --name "High-Error-Rate" \
  --resource-group <resource-group> \
  --scopes $WORKSPACE_ID \
  --condition "count 'FluentBit_CL' > 10" \
  --condition-query "
    FluentBit_CL
    | where log_s contains 'ERROR'
    | summarize AggregatedValue = count() by bin(TimeGenerated, 5m)
  " \
  --description "Alert when error count exceeds threshold" \
  --evaluation-frequency 5m \
  --window-size 15m \
  --severity 2
```

### 15. Configure Log Sampling
Sample logs to reduce volume:

```yaml
config:
  filters: |
    [FILTER]
        Name sampling
        Match kube.*
        Percentage 10  # Sample 10% of logs
```

### 16. Send Logs to Multiple Destinations
Add additional outputs:

```yaml
config:
  outputs: |
    [OUTPUT]
        Name azure
        Match kube.*
        Customer_ID ${WORKSPACE_ID}
        Shared_Key ${WORKSPACE_KEY}
        Log_Type FluentBit_CL

    [OUTPUT]
        Name stdout
        Match kube.*
        Format json_lines

    # [OUTPUT]
    #     Name es
    #     Match kube.*
    #     Host elasticsearch.logging.svc
    #     Port 9200
    #     Index fluent-bit
```

### 17. Query Advanced Log Analytics
**Top pods by log volume:**
```kql
FluentBit_CL
| where TimeGenerated > ago(1h)
| summarize LogCount = count() by k8s_pod_name_s, k8s_namespace_name_s
| order by LogCount desc
| take 20
```

**Log timeline:**
```kql
FluentBit_CL
| where TimeGenerated > ago(24h)
| summarize LogCount = count() by bin(TimeGenerated, 1h), k8s_namespace_name_s
| render timechart
```

**Error rate by application:**
```kql
FluentBit_CL
| where TimeGenerated > ago(6h)
| extend IsError = iff(log_s contains "ERROR" or log_s contains "error", 1, 0)
| summarize ErrorRate = (sum(IsError) * 100.0 / count()) by k8s_pod_name_s
| where ErrorRate > 1
| order by ErrorRate desc
```

### 18. Configure Log Rotation and Retention
Update Log Analytics retention:

```bash
az monitor log-analytics workspace update \
  --resource-group <workspace-resource-group> \
  --workspace-name <workspace-name> \
  --retention-time 90  # Days
```

### 19. Troubleshoot Fluent Bit
Check Fluent Bit status:

```bash
# View DaemonSet status
kubectl describe daemonset fluent-bit -n logging

# Check pod logs for errors
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=100

# Check configuration
kubectl exec -n logging -it $(kubectl get pod -n logging -l app.kubernetes.io/name=fluent-bit -o jsonpath='{.items[0].metadata.name}') -- cat /fluent-bit/etc/fluent-bit.conf
```

Common issues and fixes:
```bash
# If logs not appearing, check output configuration
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit | grep -i error

# Verify connectivity to Log Analytics
kubectl exec -n logging -it $(kubectl get pod -n logging -l app.kubernetes.io/name=fluent-bit -o jsonpath='{.items[0].metadata.name}') -- nc -zv ods.opinsights.azure.com 443
```

### 20. Export Logs for Compliance
Export logs from Log Analytics:

```bash
# Query and export to CSV
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "
    FluentBit_CL
    | where TimeGenerated > ago(7d)
    | project TimeGenerated, log_s, k8s_pod_name_s, k8s_namespace_name_s
  " \
  --output table > exported_logs.csv
```

## Expected Results
- Fluent Bit DaemonSet running on all nodes
- Container logs forwarded to Log Analytics
- Logs queryable with KQL in Azure Portal
- Kubernetes metadata enriched in logs
- Structured JSON logs parsed correctly
- Multi-line logs handled properly
- Alerts triggered based on log patterns
- Metrics exposed for Fluent Bit monitoring

## Cleanup
```bash
# Delete test applications
kubectl delete deployment json-logger text-logger multiline-logger

# Uninstall Fluent Bit
helm uninstall fluent-bit -n logging

# Delete namespace
kubectl delete namespace logging
```

## Key Takeaways
- **Fluent Bit** is lightweight log forwarder (vs Fluentd)
- **DaemonSet** ensures log collection from all nodes
- **Tail input** reads container logs from `/var/log/containers/`
- **Kubernetes filter** enriches logs with pod metadata
- **Azure output** sends logs to Log Analytics
- Parsers extract structured data from unstructured logs
- Filters transform and enrich log data
- Buffering provides reliability during network issues
- Multiple outputs enable sending logs to different destinations

## Fluent Bit vs Fluentd

| Feature | Fluent Bit | Fluentd |
|---------|-----------|---------|
| Memory footprint | ~450KB | ~40MB |
| Language | C | Ruby/C |
| Performance | Higher | Good |
| Plugins | 70+ | 500+ |
| Use case | Edge/aggregator | Central aggregator |
| Best for | Node-level | Cluster-level |

## Common Fluent Bit Filters

| Filter | Purpose |
|--------|---------|
| `kubernetes` | Add K8s metadata |
| `parser` | Parse log formats |
| `grep` | Include/exclude logs |
| `modify` | Add/remove fields |
| `nest` | Restructure JSON |
| `rewrite_tag` | Route based on content |

## Troubleshooting
- **No logs appearing**: Check Fluent Bit pod logs for errors
- **High memory usage**: Reduce Mem_Buf_Limit or enable buffering
- **Missing metadata**: Verify kubernetes filter is configured
- **Permission denied**: Check serviceaccount and RBAC

---


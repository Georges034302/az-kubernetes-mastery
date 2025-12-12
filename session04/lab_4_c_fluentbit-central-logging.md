# Lab 4c: Fluent Bit Central Logging
![IMG](https://github.com/user-attachments/assets/101d36eb-3a70-4c00-ac55-89abbc78cfa6)

## Objective
Forward AKS container logs to Log Analytics using Fluent Bit for centralized monitoring, enrichment, parsing, and search.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Helm 3 installed

---

## Lab Parameters

```bash
# Resource and location settings
RESOURCE_GROUP="rg-aks-fluentbit-lab"
LOCATION="australiaeast"  # Sydney, Australia

# AKS cluster settings
CLUSTER_NAME="aks-fluentbit-cluster"
NODE_COUNT=2
NODE_SIZE="Standard_D2s_v3"
K8S_VERSION="1.29"

# Log Analytics workspace settings
WORKSPACE_NAME="law-aks-fluentbit"

# Fluent Bit settings
LOGGING_NAMESPACE="logging"
FLUENTBIT_RELEASE="fluent-bit"
FLUENTBIT_CHART="fluent/fluent-bit"

# Test application settings
APP_NAMESPACE="default"
```

Display configuration:
```bash
echo "=== Lab 4c: Fluent Bit Central Logging ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Node Count: $NODE_COUNT"
echo "Node Size: $NODE_SIZE"
echo "Kubernetes Version: $K8S_VERSION"
echo "Log Analytics Workspace: $WORKSPACE_NAME"
echo "Logging Namespace: $LOGGING_NAMESPACE"
echo "Fluent Bit Release: $FLUENTBIT_RELEASE"
echo "App Namespace: $APP_NAMESPACE"
```

---

## Step 1: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \           # `Resource group name`
  --location $LOCATION                # `Azure region`
```

---

## Step 2: Create Log Analytics Workspace

```bash
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \  # `Resource group`
  --workspace-name $WORKSPACE_NAME \  # `Workspace name`
  --location $LOCATION                # `Azure region`
```

Capture workspace credentials:
```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME \
  --query customerId \
  --output tsv)

WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME \
  --query primarySharedKey \
  --output tsv)

echo "Workspace ID: $WORKSPACE_ID"
echo "Workspace Key: ${WORKSPACE_KEY:0:20}..."
```

---

## Step 3: Create AKS Cluster

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \           # `Resource group`
  --name $CLUSTER_NAME \                       # `Cluster name`
  --location $LOCATION \                       # `Azure region`
  --node-count $NODE_COUNT \                   # `Number of nodes`
  --node-vm-size $NODE_SIZE \                  # `VM size for nodes`
  --kubernetes-version $K8S_VERSION \          # `Kubernetes version`
  --network-plugin azure \                     # `Azure CNI networking`
  --generate-ssh-keys                          # `Generate SSH keys`
```

Get credentials:
```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing
```

---

## Step 4: Create Logging Namespace

```bash
kubectl create namespace $LOGGING_NAMESPACE
```

---

## Step 5: Create Secret for Log Analytics Credentials

```bash
kubectl create secret generic log-analytics-secret \
  --from-literal=workspace-id=$WORKSPACE_ID \     # `Log Analytics workspace ID`
  --from-literal=workspace-key=$WORKSPACE_KEY \   # `Log Analytics shared key`
  --namespace $LOGGING_NAMESPACE                  # `Target namespace`
```

---

## Step 6: Add Fluent Bit Helm Repository

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

---

## Step 7: Create Fluent Bit Helm Values

Create inline Helm values configuration:

```bash
cat <<EOF > fluent-bit-values.yaml
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
        Add cluster_name $CLUSTER_NAME

  outputs: |
    [OUTPUT]
        Name azure
        Match kube.*
        Customer_ID \${WORKSPACE_ID}
        Shared_Key \${WORKSPACE_KEY}
        Log_Type FluentBit_CL

    [OUTPUT]
        Name azure
        Match host.*
        Customer_ID \${WORKSPACE_ID}
        Shared_Key \${WORKSPACE_KEY}
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
EOF
```

---

## Step 8: Install Fluent Bit with Helm

```bash
helm install $FLUENTBIT_RELEASE $FLUENTBIT_CHART \
  --namespace $LOGGING_NAMESPACE \     # `Target namespace`
  --values fluent-bit-values.yaml      # `Helm values file`
```

---

## Step 9: Verify Fluent Bit Installation

```bash
kubectl logs \
  --namespace $LOGGING_NAMESPACE \
  --selector app.kubernetes.io/name=fluent-bit \
  --tail=50
```

---

## Step 10: Deploy Test Applications

Deploy JSON logger:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: json-logger
  namespace: $APP_NAMESPACE
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
            echo "{\"timestamp\":\"\$(date -Iseconds)\",\"level\":\"info\",\"message\":\"Application running\",\"request_id\":\"\$RANDOM\"}"
            sleep 5
          done
EOF
```

Deploy plain text logger:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: text-logger
  namespace: $APP_NAMESPACE
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
            echo "\$(date '+%Y-%m-%d %H:%M:%S') [INFO] Processing request \$RANDOM"
            echo "\$(date '+%Y-%m-%d %H:%M:%S') [WARN] High memory usage detected"
            echo "\$(date '+%Y-%m-%d %H:%M:%S') [ERROR] Connection timeout to database"
            sleep 10
          done
EOF
```

Deploy multi-line logger (Python tracebacks):
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multiline-logger
  namespace: $APP_NAMESPACE
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
EOF
```

---

## Step 12: Query Logs in Log Analytics

Wait 5-10 minutes for log ingestion, then query in Azure Portal or via CLI.

**Query container logs (last 30 minutes):**
```kql
FluentBit_CL
| where TimeGenerated > ago(30m)
| project TimeGenerated, log_s, k8s_pod_name_s, k8s_namespace_name_s, cluster_name_s
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
| project TimeGenerated, log_s, k8s_pod_name_s, k8s_namespace_name_s
| order by TimeGenerated desc
```

**Top pods by log volume:**
```kql
FluentBit_CL
| where TimeGenerated > ago(1h)
| summarize LogCount = count() by k8s_pod_name_s, k8s_namespace_name_s
| order by LogCount desc
| take 20
```

**Log timeline by namespace:**
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

---

## Step 13: Configure JSON Log Parsing (Optional)

Update Fluent Bit to parse JSON logs automatically:

```bash
cat <<EOF > fluent-bit-values-json-parser.yaml
config:
  service: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020

  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        multiline.parser docker, cri
        Tag kube.*
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Labels On
        Keep_Log Off

    [FILTER]
        Name parser
        Match kube.*
        Key_Name log
        Parser json
        Reserve_Data On
        Preserve_Key On

    [FILTER]
        Name modify
        Match kube.*
        Add source_type kubernetes
        Add cluster_name $CLUSTER_NAME

  outputs: |
    [OUTPUT]
        Name azure
        Match kube.*
        Customer_ID \${WORKSPACE_ID}
        Shared_Key \${WORKSPACE_KEY}
        Log_Type FluentBit_CL
        Retry_Limit 3

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

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
EOF
```

Upgrade Helm release:
```bash
helm upgrade $FLUENTBIT_RELEASE $FLUENTBIT_CHART \
  --namespace $LOGGING_NAMESPACE \
  --values fluent-bit-values-json-parser.yaml
```

---

## Step 14: Add Custom Parsers (Optional)

Create custom parser for timestamp and nginx logs:

```bash
cat <<EOF > fluent-bit-values-custom-parsers.yaml
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
        Name kubernetes
        Match kube.*
        Merge_Log On
        Labels On

    [FILTER]
        Name parser
        Match kube.*
        Key_Name log
        Parser custom_timestamp
        Reserve_Data On

    [FILTER]
        Name modify
        Match kube.*
        Add cluster_name $CLUSTER_NAME

  outputs: |
    [OUTPUT]
        Name azure
        Match kube.*
        Customer_ID \${WORKSPACE_ID}
        Shared_Key \${WORKSPACE_KEY}
        Log_Type FluentBit_CL

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
EOF
```

Apply custom parsers:
```bash
helm upgrade $FLUENTBIT_RELEASE $FLUENTBIT_CHART \
  --namespace $LOGGING_NAMESPACE \
  --values fluent-bit-values-custom-parsers.yaml
```

---

## Step 15: Exclude Namespaces from Logging (Optional)

Exclude `kube-system` namespace from log forwarding:

```bash
cat <<EOF > fluent-bit-values-exclude-ns.yaml
config:
  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Labels On

    [FILTER]
        Name grep
        Match kube.*
        Exclude k8s_namespace_name_s kube-system

    [FILTER]
        Name modify
        Match kube.*
        Add cluster_name $CLUSTER_NAME

  outputs: |
    [OUTPUT]
        Name azure
        Match kube.*
        Customer_ID \${WORKSPACE_ID}
        Shared_Key \${WORKSPACE_KEY}
        Log_Type FluentBit_CL

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
EOF
```

---

## Step 16: Monitor Fluent Bit Metrics

Expose Fluent Bit metrics endpoint:
```bash
kubectl port-forward \
  --namespace $LOGGING_NAMESPACE \
  svc/$FLUENTBIT_RELEASE 2020:2020
```

Access metrics at: `http://localhost:2020/api/v1/metrics`

Query Prometheus-format metrics:
```bash
curl http://localhost:2020/api/v1/metrics/prometheus
```

---

## Step 17: Create Log-Based Alerts

Create alert rule for high error rates:

```bash
az monitor scheduled-query create \
  --name "High-Error-Rate-Alert" \                        # `Alert rule name`
  --resource-group $RESOURCE_GROUP \                      # `Resource group`
  --scopes $(az monitor log-analytics workspace show --resource-group $RESOURCE_GROUP --workspace-name $WORKSPACE_NAME --query id -o tsv) \  # `Log Analytics workspace resource ID`
  --condition "count > 10" \                              # `Alert threshold`
  --condition-query "FluentBit_CL | where log_s contains 'ERROR' | summarize AggregatedValue = count()" \  # `KQL query`
  --description "Alert when error count exceeds 10 in 5 minutes" \  # `Alert description`
  --evaluation-frequency 5 \                              # `Evaluate every 5 minutes`
  --window-size 15 \                                      # `Query time window 15 minutes`
  --severity 2                                            # `Severity level (2 = Warning)`
```

---

## Step 18: Configure Log Sampling (Optional)

Sample logs to reduce volume (keep only 10%):

```bash
cat <<EOF > fluent-bit-values-sampling.yaml
config:
  filters: |
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Labels On

    [FILTER]
        Name sampling
        Match kube.*
        Percentage 10

    [FILTER]
        Name modify
        Match kube.*
        Add cluster_name $CLUSTER_NAME

  outputs: |
    [OUTPUT]
        Name azure
        Match kube.*
        Customer_ID \${WORKSPACE_ID}
        Shared_Key \${WORKSPACE_KEY}
        Log_Type FluentBit_CL

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
EOF
```

---

## Step 19: Configure Log Retention

Update Log Analytics workspace retention:

```bash
az monitor log-analytics workspace update \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME \
  --retention-time 90                   # `Retention in days (30-730)`
```

---

## Step 20: Export Logs for Compliance

Export logs from Log Analytics:

```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \                            # `Log Analytics workspace ID`
  --analytics-query "FluentBit_CL | where TimeGenerated > ago(7d) | project TimeGenerated, log_s, k8s_pod_name_s, k8s_namespace_name_s" \  # `KQL export query`
  --output json > exported_logs.json                     # `Export to JSON file`
```

---

## Expected Results

After completing this lab, you should have:

- ✅ Fluent Bit DaemonSet running on all AKS nodes
- ✅ Container logs forwarded to Log Analytics workspace
- ✅ Logs queryable with KQL in Azure Portal
- ✅ Kubernetes metadata (pod name, namespace, labels) enriched in logs
- ✅ JSON logs parsed into structured fields
- ✅ Multi-line logs (Python tracebacks) handled correctly
- ✅ Log-based alerts configured and functional
- ✅ Fluent Bit metrics exposed via HTTP endpoint
- ✅ Custom parsers and filters applied
- ✅ Log retention policies configured

---

## Cleanup

### Option 1: Delete Lab Resources Only

Delete test applications:
```bash
kubectl delete deployment json-logger text-logger multiline-logger \
  --namespace $APP_NAMESPACE
```

Uninstall Fluent Bit:
```bash
helm uninstall $FLUENTBIT_RELEASE \
  --namespace $LOGGING_NAMESPACE
```

Delete logging namespace:
```bash
kubectl delete namespace $LOGGING_NAMESPACE
```

Delete Log Analytics workspace:
```bash
az monitor log-analytics workspace delete \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME \
  --yes
```

### Option 2: Delete Entire Resource Group

Delete all resources including AKS cluster:
```bash
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait
```

Clean up local kubeconfig:
```bash
kubectl config delete-context $CLUSTER_NAME
```

---

## How This Connects to Azure Kubernetes Observability

### Fluent Bit Architecture in AKS

```
┌─────────────────────────────────────────────────────────────┐
│                     AKS Cluster                              │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │                  │
│  │          │  │          │  │          │                  │
│  │  ┌────┐  │  │  ┌────┐  │  │  ┌────┐  │                  │
│  │  │Pod │  │  │  │Pod │  │  │  │Pod │  │  Application     │
│  │  └────┘  │  │  └────┘  │  │  └────┘  │  Containers      │
│  │    │     │  │    │     │  │    │     │                  │
│  │    ▼     │  │    ▼     │  │    ▼     │                  │
│  │ stdout   │  │ stdout   │  │ stdout   │                  │
│  │ stderr   │  │ stderr   │  │ stderr   │                  │
│  │    │     │  │    │     │  │    │     │                  │
│  │    ▼     │  │    ▼     │  │    ▼     │                  │
│  │ /var/log/containers/*.log  │  CRI-O/  │                  │
│  │    │     │  │    │     │  │containerd│                  │
│  │    ▼     │  │    ▼     │  │    ▼     │                  │
│  │┌────────┐│  │┌────────┐│  │┌────────┐│  Fluent Bit     │
│  ││Fluent  ││  ││Fluent  ││  ││Fluent  ││  DaemonSet      │
│  ││Bit Pod ││  ││Bit Pod ││  ││Bit Pod ││  (one per node) │
│  │└────────┘│  │└────────┘│  │└────────┘│                  │
│  │    │     │  │    │     │  │    │     │                  │
│  └────┼─────┘  └────┼─────┘  └────┼─────┘                  │
│       │             │             │                         │
│       └─────────────┴─────────────┘                         │
│                     │                                        │
└─────────────────────┼────────────────────────────────────────┘
                      │
                      │ HTTPS (TLS)
                      │ Azure Output Plugin
                      ▼
        ┌──────────────────────────┐
        │  Log Analytics Workspace │
        │                          │
        │  FluentBit_CL Table      │
        │  - log_s                 │
        │  - k8s_pod_name_s        │
        │  - k8s_namespace_name_s  │
        │  - k8s_labels            │
        │  - cluster_name_s        │
        │  - TimeGenerated         │
        └──────────────────────────┘
                      │
                      ▼
        ┌──────────────────────────┐
        │   Azure Portal / KQL     │
        │   - Queries              │
        │   - Dashboards           │
        │   - Alerts               │
        └──────────────────────────┘
```

### Fluent Bit Components

| Component | Description | Configuration |
|-----------|-------------|---------------|
| **INPUT** | Source of log data | `tail` (reads `/var/log/containers/*.log`), `systemd` |
| **FILTER** | Transform/enrich logs | `kubernetes` (metadata), `parser` (JSON), `modify` (add fields), `grep` (filter) |
| **OUTPUT** | Destination for logs | `azure` (Log Analytics), `stdout`, `elasticsearch` |
| **PARSER** | Extract structured data | `json`, `regex`, `logfmt` |

### Fluent Bit vs Fluentd

| Feature | Fluent Bit | Fluentd |
|---------|-----------|---------|
| **Memory Footprint** | ~450KB | ~40MB |
| **Language** | C (compiled) | Ruby/C (interpreted) |
| **Performance** | Higher throughput | Good |
| **Plugins** | 70+ built-in | 500+ community |
| **Configuration** | INI-style | Ruby DSL |
| **Use Case** | Edge/node-level | Central aggregator |
| **Best For** | DaemonSet on nodes | Centralized processing |
| **Multi-line Support** | Built-in parsers | Requires plugins |

### Common Fluent Bit Filters

| Filter | Purpose | Example Use Case |
|--------|---------|------------------|
| `kubernetes` | Enrich with K8s metadata | Add pod name, namespace, labels |
| `parser` | Parse log formats | Extract JSON fields from log string |
| `grep` | Include/exclude logs | Filter out kube-system logs |
| `modify` | Add/remove/rename fields | Add cluster name to all logs |
| `nest` | Restructure JSON | Flatten nested Kubernetes metadata |
| `rewrite_tag` | Route based on content | Send ERROR logs to different destination |
| `throttle` | Rate limiting | Limit high-volume logs |
| `lua` | Custom transformations | Complex field manipulation |

### Log Analytics Integration

**FluentBit_CL Table Schema:**
- `log_s`: Raw log message
- `k8s_pod_name_s`: Pod name (from Kubernetes filter)
- `k8s_namespace_name_s`: Namespace (from Kubernetes filter)
- `k8s_container_name_s`: Container name
- `k8s_labels`: Pod labels (JSON)
- `cluster_name_s`: Cluster identifier (from modify filter)
- `source_type_s`: "kubernetes" (from modify filter)
- `TimeGenerated`: Log ingestion timestamp

**Query Performance Tips:**
1. Always filter by `TimeGenerated` first
2. Use `where` before `project` for efficiency
3. Index commonly queried fields
4. Use `summarize` to aggregate large result sets
5. Leverage `materialize()` for repeated subqueries

---

## Key Takeaways

- **Fluent Bit DaemonSet** ensures log collection from every node without impacting application performance
- **Tail Input** reads container logs from `/var/log/containers/` written by container runtime (containerd/CRI-O)
- **Kubernetes Filter** enriches logs with pod metadata (name, namespace, labels) via API server queries
- **Azure Output Plugin** sends logs to Log Analytics using shared key authentication
- **Parsers** extract structured data from unstructured log strings (JSON, regex patterns)
- **Multi-line Support** handles stack traces and complex log formats using `multiline.parser`
- **Buffering** provides reliability during network issues with `Retry_Limit`
- **Filters** enable transformation (add fields), filtering (exclude namespaces), and routing (conditional outputs)
- **Custom Parsers** support application-specific log formats (nginx, Apache, custom timestamp formats)
- **Multiple Outputs** allow sending logs to Log Analytics, Elasticsearch, stdout, or S3 simultaneously
- **Sampling** reduces log volume and costs while maintaining statistical accuracy
- **Log Analytics** provides centralized search, KQL queries, dashboards, and alerting
- **Metrics Endpoint** exposes Fluent Bit health and performance metrics in Prometheus format

### When to Use Fluent Bit vs Container Insights

| Scenario | Fluent Bit | Container Insights |
|----------|------------|-------------------|
| **Custom parsing** | ✅ Full control | ❌ Limited |
| **Multiple outputs** | ✅ Yes (Azure, ES, S3) | ❌ Only Log Analytics |
| **Cost optimization** | ✅ Sampling, filtering | ⚠️ Ingestion-based pricing |
| **Ease of setup** | ⚠️ Helm + configuration | ✅ One-click addon |
| **Performance metrics** | ❌ Requires separate tools | ✅ Built-in (Prometheus) |
| **Application logs** | ✅ Primary use case | ✅ Supported |
| **Resource metrics** | ❌ Not supported | ✅ CPU, memory, disk |

**Recommendation**: Use **Container Insights** for resource metrics + performance monitoring. Use **Fluent Bit** for advanced log processing, custom parsing, and multi-destination forwarding.

---

## Troubleshooting

### No Logs Appearing in Log Analytics

Check Fluent Bit pod logs:
```bash
kubectl logs \
  --namespace $LOGGING_NAMESPACE \
  --selector app.kubernetes.io/name=fluent-bit \
  --tail=100
```

### High Memory Usage

**Reduce memory buffer limit:**
```yaml
Mem_Buf_Limit 1MB  # Default is 5MB
```

**Enable disk buffering:**
```yaml
[INPUT]
    Name tail
    Path /var/log/containers/*.log
    DB /var/log/fluentbit.db
```

### Missing Kubernetes Metadata

Verify RBAC permissions:
```bash
kubectl get clusterrolebinding | grep fluent-bit
```

### Logs Not Parsed Correctly

Check Fluent Bit parser configuration in Helm values and verify JSON format matches parser regex patterns.

---


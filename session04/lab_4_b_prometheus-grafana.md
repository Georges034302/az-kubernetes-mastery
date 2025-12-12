# Lab 4b: Prometheus & Grafana Monitoring
<img width="800" height="407" alt="ZIMAGE" src="https://github.com/user-attachments/assets/99f0cc7f-af51-40f8-a483-d952966980f1" />

## Objective
Deploy **Prometheus** and **Grafana** using the **kube-prometheus-stack** Helm chart to collect, store, and visualize AKS cluster metrics, create custom dashboards, and configure alerting rules.

---

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Helm 3 installed
- Subscription with permissions to create resources
- Basic understanding of Prometheus and PromQL

---

## 1. Define Environment Variables

```bash
# Resource naming
RESOURCE_GROUP="rg-aks-prometheus"
LOCATION="australiaeast"
CLUSTER_NAME="aks-prometheus"
MONITORING_NAMESPACE="monitoring"
APP_NAMESPACE="demo-app"

# AKS configuration
NODE_COUNT=3
NODE_SIZE="Standard_DS2_v2"
K8S_VERSION="1.29"

# Prometheus configuration
PROMETHEUS_RETENTION="7d"
PROMETHEUS_STORAGE="10Gi"
GRAFANA_PASSWORD="SecureP@ssw0rd123"
```

Display variables:
```bash
echo "RESOURCE_GROUP=$RESOURCE_GROUP"
echo "LOCATION=$LOCATION"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "MONITORING_NAMESPACE=$MONITORING_NAMESPACE"
echo "APP_NAMESPACE=$APP_NAMESPACE"
echo "NODE_COUNT=$NODE_COUNT"
echo "NODE_SIZE=$NODE_SIZE"
echo "K8S_VERSION=$K8S_VERSION"
echo "PROMETHEUS_RETENTION=$PROMETHEUS_RETENTION"
echo "PROMETHEUS_STORAGE=$PROMETHEUS_STORAGE"
echo "GRAFANA_PASSWORD=$GRAFANA_PASSWORD"
```

---

## 2. Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP `# Target resource group` \
  --location $LOCATION `# Azure region`
```

---

## 3. Create AKS Cluster

```bash
az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --kubernetes-version $K8S_VERSION `# Kubernetes version` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_SIZE `# Node VM size` \
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

## 4. Create Namespaces

```bash
kubectl create namespace $MONITORING_NAMESPACE
kubectl create namespace $APP_NAMESPACE
```

---

## 5. Install kube-prometheus-stack with Helm

Add Prometheus community Helm repository:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install kube-prometheus-stack:
```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace $MONITORING_NAMESPACE `# Target namespace` \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false `# Allow all ServiceMonitors` \
  --set prometheus.prometheusSpec.retention=$PROMETHEUS_RETENTION `# Metrics retention period` \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=$PROMETHEUS_STORAGE `# Storage size` \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=default `# Storage class` \
  --set grafana.adminPassword=$GRAFANA_PASSWORD `# Grafana admin password` \
  --set grafana.service.type=LoadBalancer `# Expose Grafana via LoadBalancer`
```

---

## 6. Verify Installation

Wait for all pods to be ready:
```bash
kubectl wait pod \
  --namespace $MONITORING_NAMESPACE `# Monitoring namespace` \
  --selector release=prometheus `# Prometheus stack release` \
  --for=condition=Ready `# Wait for Ready condition` \
  --timeout=600s `# Timeout after 10 minutes`
```

---

## 7. Access Grafana Dashboard

Retrieve Grafana LoadBalancer IP:
```bash
GRAFANA_IP=$(kubectl get service prometheus-grafana \
  --namespace $MONITORING_NAMESPACE \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "GRAFANA_IP=$GRAFANA_IP"
```

Display Grafana access information:
```bash
echo "Grafana URL: http://$GRAFANA_IP"
echo "Username: admin"
echo "Password: $GRAFANA_PASSWORD"
```

---

## 8. Access Prometheus UI (Port-Forward)

Port-forward to Prometheus:
```bash
kubectl port-forward \
  --namespace $MONITORING_NAMESPACE `# Monitoring namespace` \
  service/prometheus-kube-prometheus-prometheus `# Prometheus service` \
  9090:9090 `# Local port:service port`
```

**Access at**: http://localhost:9090

---

## 9. Deploy Sample Application with Metrics Endpoint

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: $APP_NAMESPACE
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: quay.io/brancz/prometheus-example-app:v0.3.0
        ports:
        - containerPort: 8080
          name: metrics
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: $APP_NAMESPACE
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - port: 8080
    targetPort: 8080
    name: metrics
EOF
```

Wait for deployment:
```bash
kubectl rollout status deployment/sample-app --namespace $APP_NAMESPACE
```

---

## 10. Create ServiceMonitor for Custom Application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app-monitor
  namespace: $APP_NAMESPACE
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: sample-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
EOF
```

**Note**: Verify scrape target appears in Prometheus UI → Status → Targets

---

## 11. Query Prometheus Metrics

### Essential PromQL Queries

**CPU usage by pod**:
```promql
rate(container_cpu_usage_seconds_total{namespace!=""}[5m])
```

**Memory usage by namespace**:
```promql
sum(container_memory_usage_bytes{namespace!=""}) by (namespace)
```

**Node CPU usage**:
```promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Pod count by namespace**:
```promql
count(kube_pod_info) by (namespace)
```

**HTTP request rate (sample app)**:
```promql
rate(http_requests_total{namespace="$APP_NAMESPACE"}[5m])
```

**95th percentile latency (sample app)**:
```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

---

## 12. Create PrometheusRule for Alerting

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: $MONITORING_NAMESPACE
  labels:
    release: prometheus
spec:
  groups:
  - name: pod-alerts
    interval: 30s
    rules:
    - alert: HighPodMemory
      expr: |
        sum(container_memory_usage_bytes{namespace!="kube-system"}) by (pod, namespace) 
        / sum(container_spec_memory_limit_bytes{namespace!="kube-system"}) by (pod, namespace) 
        > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ \$labels.namespace }}/{{ \$labels.pod }} high memory usage"
        description: "Pod is using {{ \$value | humanizePercentage }} of memory limit"
    
    - alert: PodRestartingTooMuch
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ \$labels.namespace }}/{{ \$labels.pod }} restarting"
        description: "Pod has restarted {{ \$value }} times in 15 minutes"
    
    - alert: HighCPUUsage
      expr: |
        sum(rate(container_cpu_usage_seconds_total{namespace!="kube-system"}[5m])) by (pod, namespace)
        / sum(container_spec_cpu_quota{namespace!="kube-system"} / container_spec_cpu_period{namespace!="kube-system"}) by (pod, namespace)
        > 0.9
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "High CPU usage on {{ \$labels.namespace }}/{{ \$labels.pod }}"
        description: "CPU usage is {{ \$value | humanizePercentage }}"
EOF
```

**Note**: Verify alerts appear in Prometheus UI → Alerts tab

---

## 13. Access Alertmanager (Port-Forward)

Port-forward to Alertmanager:
```bash
kubectl port-forward \
  --namespace $MONITORING_NAMESPACE `# Monitoring namespace` \
  service/prometheus-kube-prometheus-alertmanager `# Alertmanager service` \
  9093:9093 `# Local port:service port`
```

**Access at**: http://localhost:9093

---

## 14. Advanced PromQL Queries

### Top 10 Pods by Memory
```promql
topk(10, sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace))
```

### Pod Network Bandwidth
```promql
sum(rate(container_network_transmit_bytes_total[5m])) by (pod, namespace)
```

### Container Restart Count
```promql
sum(kube_pod_container_status_restarts_total) by (namespace, pod)
```

### Persistent Volume Usage
```promql
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100
```

---

## 15. Cleanup

**Option 1**: Remove application and monitoring stack:
```bash
kubectl delete namespace $APP_NAMESPACE

helm uninstall prometheus --namespace $MONITORING_NAMESPACE

kubectl delete pvc \
  --namespace $MONITORING_NAMESPACE \
  --selector app.kubernetes.io/name=prometheus

kubectl delete namespace $MONITORING_NAMESPACE
```

**Option 2**: Delete entire resource group:
```bash
az group delete \
  --name $RESOURCE_GROUP `# Resource group to delete` \
  --yes `# Skip confirmation` \
  --no-wait `# Run asynchronously`
```

---

## How This Connects to Kubernetes Monitoring

### kube-prometheus-stack Architecture
```
        ┌──────────────────────────────┐
        │   AKS Cluster                │
        │  ┌────────────────────────┐  │
        │  │  Application Pods      │  │
        │  │  • Expose /metrics     │  │
        │  │  • Port 8080           │  │
        │  └─────────┬──────────────┘  │
        │            │                  │
        │  ┌─────────▼──────────────┐  │
        │  │  Prometheus Server     │  │
        │  │  • Scrapes targets     │  │
        │  │  • Stores TSDB         │  │
        │  │  • Evaluates rules     │  │
        │  └─────────┬──────────────┘  │
        │            │                  │
        │  ┌─────────▼──────────────┐  │
        │  │  Alertmanager          │  │
        │  │  • Routes alerts       │  │
        │  │  • Deduplication       │  │
        │  │  • Notifications       │  │
        │  └────────────────────────┘  │
        └──────────────┬───────────────┘
                       │
        ┌──────────────▼───────────────┐
        │  Grafana Dashboard           │
        │  • Visualizes metrics        │
        │  • Custom dashboards         │
        │  • Pre-built dashboards      │
        └──────────────────────────────┘
```

### Key Components

| Component | Purpose | Configuration |
|-----------|---------|---------------|
| **Prometheus** | Time-series database, scrapes metrics | ServiceMonitor, PrometheusRule |
| **Grafana** | Visualization and dashboarding | Dashboards, datasources |
| **Alertmanager** | Alert routing and notification | Alertmanager config secret |
| **ServiceMonitor** | Defines scrape targets | CRD with label selector |
| **PrometheusRule** | Defines alerts and recording rules | CRD with PromQL expressions |
| **Node Exporter** | Exposes node-level metrics | DaemonSet (auto-installed) |
| **kube-state-metrics** | Exposes Kubernetes object metrics | Deployment (auto-installed) |

### PromQL Query Patterns

| Query Type | Example | Use Case |
|------------|---------|----------|
| **Rate** | `rate(http_requests_total[5m])` | Requests per second |
| **Sum** | `sum(container_memory_usage_bytes) by (namespace)` | Total memory by namespace |
| **Avg** | `avg(node_cpu_seconds_total) by (mode)` | Average CPU per mode |
| **Topk** | `topk(10, container_memory_usage_bytes)` | Top 10 memory consumers |
| **Quantile** | `histogram_quantile(0.95, rate(duration_bucket[5m]))` | 95th percentile latency |
| **Count** | `count(kube_pod_info) by (namespace)` | Pod count per namespace |

### ServiceMonitor vs PodMonitor

| Aspect | ServiceMonitor | PodMonitor |
|--------|----------------|------------|
| **Targets** | Services | Pods directly |
| **Selector** | Service labels | Pod labels |
| **Use Case** | Stable endpoints | Dynamic pods |
| **Discovery** | Via Service | Via Pod |

### Alerting Workflow

```
Prometheus → Evaluates PromQL → Fires Alert → Alertmanager
                                                    ↓
                                        Routes by label/receiver
                                                    ↓
                                    ┌───────────────┴──────────────┐
                                    ↓                              ↓
                            Webhook/Slack                    Email/PagerDuty
```

### Grafana Dashboard Components

| Component | Description | Example |
|-----------|-------------|---------|
| **Panel** | Single visualization | Line chart, gauge, table |
| **Variable** | Dynamic filter | Namespace selector |
| **Row** | Groups related panels | CPU metrics row |
| **Annotation** | Event marker | Deployment timestamp |
| **Alert** | Panel-based alert | High CPU threshold |

### Pre-installed Dashboards

The kube-prometheus-stack includes comprehensive dashboards:

- **Kubernetes / Compute Resources / Cluster** - Cluster-wide CPU/memory/network
- **Kubernetes / Compute Resources / Namespace (Pods)** - Per-namespace pod metrics
- **Kubernetes / Compute Resources / Node (Pods)** - Node-level resource allocation
- **Kubernetes / Networking / Cluster** - Network I/O and bandwidth
- **Kubernetes / Persistent Volumes** - PV/PVC usage and capacity

---

## Expected Results

✅ **AKS cluster deployed** with sufficient resources  
✅ **kube-prometheus-stack installed** via Helm  
✅ **Prometheus** scraping cluster and application metrics  
✅ **Grafana** accessible via LoadBalancer IP with pre-built dashboards  
✅ **Sample application** deployed with `/metrics` endpoint  
✅ **ServiceMonitor** created and targets appearing in Prometheus  
✅ **PrometheusRule** created with custom alerts  
✅ **Alertmanager** accessible and ready for notification routing  
✅ **PromQL queries** executed successfully for various metrics  
✅ **Persistent storage** allocated for Prometheus TSDB  

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| **No metrics in Grafana** | ServiceMonitor not created | Verify ServiceMonitor with matching labels |
| **Alerts not firing** | Wrong `release` label | Ensure PrometheusRule has `release: prometheus` label |
| **Prometheus high memory** | Long retention period | Reduce retention or increase resources |
| **Missing scrape targets** | Label selector mismatch | Check ServiceMonitor selector matches Service labels |
| **Grafana LoadBalancer pending** | No external IP available | Use port-forward or NodePort instead |
| **Pods not ready** | Insufficient cluster resources | Scale up nodes or reduce replica counts |
| **Storage errors** | No default StorageClass | Specify storageClassName explicitly |

---

## Key Takeaways

- **kube-prometheus-stack** provides complete monitoring solution (Prometheus + Grafana + Alertmanager)
- **Prometheus** uses pull-based model, scraping `/metrics` endpoints via HTTP
- **ServiceMonitor** CRD defines which services to scrape
- **PrometheusRule** CRD defines alerting rules and recording rules
- **PromQL** is powerful query language for time-series data
- **Grafana** provides rich visualization with pre-built Kubernetes dashboards
- **Alertmanager** handles alert routing, grouping, and notification
- **Labels** enable flexible metric aggregation and filtering
- **Time-series database (TSDB)** requires persistent storage for retention
- **Recording rules** pre-compute expensive queries for dashboard performance
- **Node Exporter** and **kube-state-metrics** provide essential cluster metrics

---


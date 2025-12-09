# Lab 4b: Prometheus & Grafana

## Objective
Deploy Prometheus & Grafana and visualize cluster metrics.

## Prerequisites
- AKS cluster running
- Helm 3 installed
- `kubectl` configured
- Sufficient cluster resources (CPU/memory)

## Steps

### 1. Create Monitoring Namespace
```bash
kubectl create namespace monitoring
```

### 2. Install Prometheus Stack with Helm
Add Prometheus community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install kube-prometheus-stack (includes Prometheus, Grafana, Alertmanager):
```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --set grafana.adminPassword=admin123 \
  --set grafana.service.type=LoadBalancer
```

### 3. Verify Installation
```bash
# Check all pods are running
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod -l "release=prometheus" -n monitoring --timeout=300s
```

### 4. Access Grafana Dashboard
Get Grafana LoadBalancer IP:

```bash
kubectl get service prometheus-grafana -n monitoring --watch
```

Once EXTERNAL-IP is assigned:
```bash
GRAFANA_IP=$(kubectl get service prometheus-grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Grafana URL: http://$GRAFANA_IP"
echo "Username: admin"
echo "Password: admin123"
```

Access via browser:
```bash
"$BROWSER" http://$GRAFANA_IP
```

### 5. Access Prometheus UI
Port-forward to Prometheus:

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Access at: http://localhost:9090

### 6. Explore Pre-installed Dashboards
In Grafana, navigate to Dashboards. Pre-installed dashboards include:

- **Kubernetes / Compute Resources / Cluster**
- **Kubernetes / Compute Resources / Namespace (Pods)**
- **Kubernetes / Compute Resources / Node (Pods)**
- **Kubernetes / Networking / Cluster**
- **Kubernetes / Persistent Volumes**

### 7. Query Prometheus Metrics
In Prometheus UI (http://localhost:9090), try these queries:

**CPU usage by pod:**
```promql
rate(container_cpu_usage_seconds_total{namespace!=""}[5m])
```

**Memory usage by namespace:**
```promql
sum(container_memory_usage_bytes{namespace!=""}) by (namespace)
```

**Pod count by namespace:**
```promql
count(kube_pod_info) by (namespace)
```

**Node CPU usage:**
```promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Available memory:**
```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
```

### 8. Deploy Sample Application with Metrics
Create an app that exposes Prometheus metrics:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
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
  namespace: default
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - port: 8080
    targetPort: 8080
    name: metrics
```

Apply:
```bash
kubectl apply -f sample-app.yaml
```

### 9. Create ServiceMonitor for Custom App
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app-monitor
  namespace: default
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
```

Apply:
```bash
kubectl apply -f servicemonitor.yaml
```

Verify in Prometheus UI:
- Status > Targets
- Look for "default/sample-app-monitor"

### 10. Create Custom Grafana Dashboard
In Grafana:

1. Click **+** > **Dashboard** > **Add new panel**
2. Add query:
   ```promql
   rate(http_requests_total{namespace="default"}[5m])
   ```
3. Set panel title: "HTTP Request Rate"
4. Add another panel with:
   ```promql
   histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
   ```
5. Title: "95th Percentile Response Time"
6. Save dashboard as "Sample App Metrics"

### 11. Configure Alerting Rules
Create PrometheusRule:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
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
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} high memory usage"
        description: "Pod is using {{ $value | humanizePercentage }} of memory limit"
    
    - alert: PodRestartingTooMuch
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarting"
        description: "Pod has restarted {{ $value }} times in 15 minutes"
    
    - alert: HighCPUUsage
      expr: |
        sum(rate(container_cpu_usage_seconds_total{namespace!="kube-system"}[5m])) by (pod, namespace)
        / sum(container_spec_cpu_quota{namespace!="kube-system"} / container_spec_cpu_period{namespace!="kube-system"}) by (pod, namespace)
        > 0.9
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "High CPU usage on {{ $labels.namespace }}/{{ $labels.pod }}"
        description: "CPU usage is {{ $value | humanizePercentage }}"
```

Apply:
```bash
kubectl apply -f prometheus-rules.yaml
```

Verify alerts in Prometheus:
- Alerts tab
- Check firing/pending alerts

### 12. Access Alertmanager
Port-forward to Alertmanager:

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
```

Access at: http://localhost:9093

### 13. Configure Alertmanager Notifications
Edit Alertmanager config:

```bash
kubectl edit secret alertmanager-prometheus-kube-prometheus-alertmanager -n monitoring
```

Or create new config:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-prometheus-kube-prometheus-alertmanager
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default'
    receivers:
    - name: 'default'
      webhook_configs:
      - url: 'http://webhook-receiver:8080/alerts'
        send_resolved: true
    # - name: 'email'
    #   email_configs:
    #   - to: 'alerts@example.com'
    #     from: 'prometheus@example.com'
    #     smarthost: 'smtp.example.com:587'
```

### 14. Create Custom Metrics Dashboard
Create dashboard JSON or via UI with panels:

**Panel 1: Cluster CPU Usage**
```promql
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace)
```

**Panel 2: Cluster Memory Usage**
```promql
sum(container_memory_working_set_bytes{container!=""}) by (namespace)
```

**Panel 3: Network I/O**
```promql
sum(rate(container_network_receive_bytes_total[5m])) by (namespace)
```

**Panel 4: Pod Count**
```promql
count(kube_pod_info) by (namespace)
```

**Panel 5: Disk Usage**
```promql
sum(kubelet_volume_stats_used_bytes) by (namespace) / sum(kubelet_volume_stats_capacity_bytes) by (namespace) * 100
```

### 15. Import Community Dashboards
In Grafana:

1. Click **+** > **Import**
2. Enter dashboard ID from [grafana.com](https://grafana.com/grafana/dashboards/):
   - **315**: Kubernetes cluster monitoring
   - **6417**: Kubernetes Deployment Statefulset Daemonset metrics
   - **8588**: Kubernetes Deployment metrics
3. Click **Load** and **Import**

### 16. Query Advanced Metrics
**Top 10 pods by memory:**
```promql
topk(10, sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace))
```

**Pod network bandwidth:**
```promql
sum(rate(container_network_transmit_bytes_total[5m])) by (pod, namespace)
```

**Container restart count:**
```promql
sum(kube_pod_container_status_restarts_total) by (namespace, pod)
```

**Persistent volume usage:**
```promql
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100
```

### 17. Enable Recording Rules for Performance
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: recording-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
  - name: performance
    interval: 30s
    rules:
    - record: namespace:container_cpu_usage:sum
      expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)
    
    - record: namespace:container_memory_usage:sum
      expr: sum(container_memory_working_set_bytes) by (namespace)
```

### 18. Monitor Prometheus Itself
Check Prometheus metrics:
```promql
# Scrape duration
prometheus_target_interval_length_seconds

# Ingestion rate
rate(prometheus_tsdb_head_samples_appended_total[5m])

# Storage size
prometheus_tsdb_storage_blocks_bytes
```

### 19. Configure Data Retention
Edit Prometheus spec:

```bash
kubectl edit prometheus prometheus-kube-prometheus-prometheus -n monitoring
```

Change retention:
```yaml
spec:
  retention: 15d  # or 30d
  retentionSize: "50GB"
```

### 20. Export Metrics for Analysis
Use Prometheus HTTP API:

```bash
# Query current values
curl 'http://localhost:9090/api/v1/query?query=up'

# Query range
curl 'http://localhost:9090/api/v1/query_range?query=up&start=2024-12-09T00:00:00Z&end=2024-12-09T23:59:59Z&step=1h'
```

## Expected Results
- Prometheus successfully scraping cluster metrics
- Grafana dashboards showing real-time visualizations
- Custom ServiceMonitors collecting app metrics
- Alerts firing based on defined rules
- Alertmanager routing notifications
- Persistent storage for metrics retention
- Pre-built dashboards for cluster monitoring

## Cleanup
```bash
# Delete sample app
kubectl delete -f sample-app.yaml
kubectl delete servicemonitor sample-app-monitor -n default

# Uninstall Prometheus stack
helm uninstall prometheus -n monitoring

# Delete PVCs
kubectl delete pvc -n monitoring -l app.kubernetes.io/name=prometheus

# Delete namespace
kubectl delete namespace monitoring
```

## Key Takeaways
- **kube-prometheus-stack** provides complete monitoring solution
- **ServiceMonitor** CRD defines scrape targets
- **PrometheusRule** CRD defines alerts and recording rules
- **Grafana** visualizes Prometheus metrics
- **Alertmanager** handles alert routing and notifications
- PromQL is the query language for Prometheus
- Labels enable flexible metric aggregation
- Recording rules pre-compute expensive queries
- Persistent storage required for long-term retention

## Useful PromQL Queries

| Metric | Query |
|--------|-------|
| CPU usage | `rate(container_cpu_usage_seconds_total[5m])` |
| Memory usage | `container_memory_working_set_bytes` |
| Network I/O | `rate(container_network_receive_bytes_total[5m])` |
| Disk usage | `kubelet_volume_stats_used_bytes` |
| Pod count | `count(kube_pod_info)` |
| Request rate | `rate(http_requests_total[5m])` |

## Grafana Tips
- Use variables for dynamic dashboards (namespace, pod)
- Set appropriate refresh intervals (5s, 10s, 1m)
- Use annotations to mark deployments
- Create dashboard folders for organization
- Export dashboards as JSON for version control

## Troubleshooting
- **No metrics**: Check ServiceMonitor selector matches service labels
- **Alerts not firing**: Verify PrometheusRule has correct `release` label
- **High memory usage**: Reduce retention or increase Prometheus resources
- **Missing targets**: Check pod annotations for scraping

---


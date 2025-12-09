# Lab 5d: Kiali and Jaeger Observability

## Objective
Visualize service mesh topology with Kiali and trace requests with Jaeger.

## Prerequisites
- AKS cluster running
- Istio installed (from Lab 5c)
- `kubectl` configured
- Sample application deployed in mesh

## Steps

### 1. Install Kiali
```bash
# Install Kiali operator
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# Wait for Kiali to be ready
kubectl rollout status deployment/kiali -n istio-system

# Verify installation
kubectl get pods -n istio-system -l app=kiali
kubectl get svc -n istio-system -l app=kiali
```

### 2. Access Kiali Dashboard
```bash
# Port forward to Kiali
kubectl port-forward svc/kiali -n istio-system 20001:20001

# Access dashboard at: http://localhost:20001
# Default credentials: admin/admin (if auth enabled)
```

Alternative - using istioctl:
```bash
istioctl dashboard kiali
```

### 3. Install Jaeger
```bash
# Install Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# Wait for Jaeger to be ready
kubectl rollout status deployment/jaeger -n istio-system

# Verify installation
kubectl get pods -n istio-system -l app=jaeger
```

### 4. Access Jaeger UI
```bash
# Port forward to Jaeger
kubectl port-forward svc/tracing -n istio-system 16686:80

# Access UI at: http://localhost:16686
```

Alternative:
```bash
istioctl dashboard jaeger
```

### 5. Install Prometheus and Grafana
```bash
# Install Prometheus (Kiali dependency)
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml

# Install Grafana for metrics visualization
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml

# Wait for deployments
kubectl rollout status deployment/prometheus -n istio-system
kubectl rollout status deployment/grafana -n istio-system
```

### 6. Deploy Multi-Service Application
Create `microservices-app.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability-demo
  labels:
    istio-injection: enabled
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: observability-demo
spec:
  selector:
    app: frontend
  ports:
  - name: http
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: observability-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      version: v1
  template:
    metadata:
      labels:
        app: frontend
        version: v1
    spec:
      containers:
      - name: frontend
        image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: "productcatalog:3550"
        - name: CURRENCY_SERVICE_ADDR
          value: "currency:7000"
        - name: CART_SERVICE_ADDR
          value: "cart:7070"
        - name: RECOMMENDATION_SERVICE_ADDR
          value: "recommendation:8080"
        - name: SHIPPING_SERVICE_ADDR
          value: "shipping:50051"
        - name: CHECKOUT_SERVICE_ADDR
          value: "checkout:5050"
        - name: AD_SERVICE_ADDR
          value: "adservice:9555"
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: productcatalog
  namespace: observability-demo
spec:
  selector:
    app: productcatalog
  ports:
  - name: grpc
    port: 3550
    targetPort: 3550
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: productcatalog
  namespace: observability-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: productcatalog
  template:
    metadata:
      labels:
        app: productcatalog
        version: v1
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/productcatalogservice:v0.8.0
        ports:
        - containerPort: 3550
        env:
        - name: PORT
          value: "3550"
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: recommendation
  namespace: observability-demo
spec:
  selector:
    app: recommendation
  ports:
  - name: grpc
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation
  namespace: observability-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: recommendation
  template:
    metadata:
      labels:
        app: recommendation
        version: v1
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/recommendationservice:v0.8.0
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: "productcatalog:3550"
        resources:
          requests:
            cpu: 100m
            memory: 220Mi
          limits:
            cpu: 200m
            memory: 450Mi
```

Deploy:
```bash
kubectl apply -f microservices-app.yaml

# Verify pods have sidecars (2/2)
kubectl get pods -n observability-demo

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod --all -n observability-demo --timeout=300s
```

### 7. Create Istio Gateway for Application
Create `gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: frontend-gateway
  namespace: observability-demo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: frontend
  namespace: observability-demo
spec:
  hosts:
  - "*"
  gateways:
  - frontend-gateway
  http:
  - route:
    - destination:
        host: frontend
        port:
          number: 8080
```

Apply:
```bash
kubectl apply -f gateway.yaml

# Get gateway URL
export GATEWAY_URL=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Application URL: http://$GATEWAY_URL"
```

### 8. Generate Traffic for Observability
```bash
# Generate continuous traffic
while true; do
  curl -s http://$GATEWAY_URL > /dev/null
  echo "Request sent at $(date)"
  sleep 1
done
```

Run in background:
```bash
# Install hey for load generation
go install github.com/rakyll/hey@latest

# Generate load
hey -z 10m -c 5 -q 1 http://$GATEWAY_URL
```

### 9. Explore Kiali - Service Graph
Open Kiali dashboard and explore:

**Graph View:**
- Navigate to **Graph** → Select **observability-demo** namespace
- View service topology
- Change display to show:
  - Traffic animation
  - Request percentage
  - Response time
  - Security (mTLS status)

**Graph Display Options:**
- Versioned app graph
- Workload graph
- Service graph
- Operation nodes

**Graph Settings:**
- Enable "Traffic Animation"
- Enable "Service Nodes"
- Show "Response Time" edge labels
- Show "Security" badges

### 10. Kiali - Application Metrics
In Kiali dashboard:

**Applications View:**
- Navigate to **Applications** → **observability-demo**
- Select **frontend** application
- View metrics:
  - Request volume
  - Request duration
  - Request size
  - Response size
  - TCP traffic

**Inbound/Outbound Metrics:**
- Inbound metrics (requests to this service)
- Outbound metrics (requests from this service)

### 11. Kiali - Workload Analysis
**Workloads View:**
- Navigate to **Workloads** → **observability-demo**
- Select **frontend** workload
- View:
  - Pod status
  - Container logs
  - Envoy logs
  - Envoy config

**Pod Details:**
```bash
# View pod logs directly from Kiali UI
# Or via kubectl:
POD=$(kubectl get pod -n observability-demo -l app=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n observability-demo $POD -c frontend
kubectl logs -n observability-demo $POD -c istio-proxy
```

### 12. Kiali - Service Health
**Services View:**
- Navigate to **Services** → **observability-demo**
- View health indicators:
  - Green: Healthy (error rate < 0.1%)
  - Orange: Degraded (error rate 0.1-20%)
  - Red: Failure (error rate > 20%)

**Service Metrics:**
- Request rate
- Error rate
- Duration (P50, P95, P99)

### 13. Configure Kiali Custom Dashboards
Create custom dashboard configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kiali
  namespace: istio-system
  labels:
    app: kiali
data:
  config.yaml: |
    server:
      port: 20001
      web_root: /kiali
    external_services:
      prometheus:
        url: http://prometheus:9090
      grafana:
        enabled: true
        in_cluster_url: http://grafana:3000
        url: http://localhost:3000
      tracing:
        enabled: true
        in_cluster_url: http://tracing:80
        url: http://localhost:16686
    istio_namespace: istio-system
```

### 14. Jaeger - Distributed Tracing
Open Jaeger UI and explore:

**Search Traces:**
- Service: **frontend.observability-demo**
- Operation: **all**
- Lookback: **1h**
- Click **Find Traces**

**Trace Details:**
- Click on a trace to see:
  - Span timeline
  - Service calls
  - Duration breakdown
  - Tags and logs

### 15. Analyze Trace Spans
In Jaeger, examine trace details:

**Span Information:**
- Operation name
- Start time and duration
- Tags (HTTP method, status code, etc.)
- Logs (events within span)
- Process information

**Trace DAG:**
- View trace as directed acyclic graph
- See parallel vs sequential calls
- Identify bottlenecks

### 16. Compare Traces
**Trace Comparison:**
- Select multiple traces
- Click **Compare**
- Analyze differences in:
  - Total duration
  - Span count
  - Service calls
  - Error patterns

### 17. Configure Sampling Rate
Adjust trace sampling:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
data:
  mesh: |
    defaultConfig:
      tracing:
        sampling: 100.0  # 100% sampling for demo
        zipkin:
          address: jaeger-collector.istio-system:9411
```

Apply:
```bash
kubectl apply -f istio-tracing-config.yaml

# Restart workloads to pick up new config
kubectl rollout restart deployment -n observability-demo
```

### 18. Custom Span Tags
Add custom tags to traces in application code:

```python
# Python example with OpenTelemetry
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process_request") as span:
    span.set_attribute("user.id", user_id)
    span.set_attribute("order.total", order_total)
    # Process request
```

### 19. Grafana - Istio Dashboards
Access Grafana:

```bash
kubectl port-forward svc/grafana -n istio-system 3000:3000
# Open http://localhost:3000
```

Import Istio dashboards:
- **Istio Mesh Dashboard**: Overall mesh metrics
- **Istio Service Dashboard**: Per-service metrics
- **Istio Workload Dashboard**: Per-workload metrics
- **Istio Performance Dashboard**: Control plane metrics

### 20. Create Custom Kiali Namespace Labels
Label namespaces for better organization:

```bash
# Add custom labels for Kiali filtering
kubectl label namespace observability-demo env=production
kubectl label namespace observability-demo team=platform

# View in Kiali namespace selector
```

Create custom metric queries:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kiali
  namespace: istio-system
data:
  config.yaml: |
    server:
      metrics_enabled: true
      metrics_port: 9090
    kubernetes_config:
      custom_labels:
        namespaces:
        - env
        - team
```

## Expected Results
- Kiali showing service mesh topology
- Real-time traffic flow visualization
- Service health indicators (green/orange/red)
- Request metrics (rate, duration, size)
- Jaeger displaying distributed traces
- Trace spans showing service dependencies
- End-to-end request latency breakdown
- Grafana dashboards with Istio metrics
- mTLS status visible in service graph

## Key Takeaways
- **Kiali** provides service mesh observability UI
- **Service graph** visualizes mesh topology
- **Traffic animation** shows real-time request flow
- **Health indicators** quickly identify issues
- **Jaeger** enables distributed tracing
- **Traces** show end-to-end request path
- **Spans** represent individual service calls
- **Sampling** controls trace collection rate
- **Grafana** complements with detailed metrics
- Integration between Kiali, Jaeger, Prometheus, Grafana

## Kiali Features

| Feature | Purpose |
|---------|---------|
| Graph | Service topology visualization |
| Applications | Application-level metrics |
| Workloads | Deployment/pod details |
| Services | Service health and metrics |
| Istio Config | Validate mesh configuration |
| Distributed Tracing | Integration with Jaeger |

## Jaeger Components

| Component | Purpose |
|-----------|---------|
| Agent | Collects spans from apps |
| Collector | Receives and processes traces |
| Query | Serves UI and API |
| Storage | Stores trace data |

## Troubleshooting
- **No data in Kiali**: Check Prometheus connection
- **Traces not appearing**: Verify sampling rate
- **Graph empty**: Ensure traffic is flowing
- **mTLS not showing**: Check PeerAuthentication
- **Slow dashboard**: Reduce time range or sampling

---


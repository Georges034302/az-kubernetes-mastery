# Lab 5c: Istio Service Mesh Setup

## Objective
Set up Istio service mesh on AKS for traffic management and observability.

## Prerequisites
- AKS cluster running
- `kubectl` configured
- Helm 3 installed
- Cluster with at least 8GB memory available

## Steps

### 1. Download and Install Istio
```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -

# Move to Istio directory
cd istio-*

# Add istioctl to PATH
export PATH=$PWD/bin:$PATH

# Verify installation
istioctl version
```

### 2. Install Istio on AKS
```bash
# Install Istio with demo profile
istioctl install --set profile=demo -y

# For production, use:
# istioctl install --set profile=production -y

# Verify installation
kubectl get pods -n istio-system

# Check Istio components
kubectl get svc -n istio-system
```

### 3. Enable Automatic Sidecar Injection
```bash
# Label namespace for automatic injection
kubectl create namespace demo
kubectl label namespace demo istio-injection=enabled

# Verify label
kubectl get namespace demo --show-labels
```

### 4. Deploy Sample Application
Create `bookinfo-app.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: productpage
  namespace: demo
  labels:
    app: productpage
    service: productpage
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: productpage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: productpage-v1
  namespace: demo
  labels:
    app: productpage
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: productpage
      version: v1
  template:
    metadata:
      labels:
        app: productpage
        version: v1
    spec:
      serviceAccountName: bookinfo-productpage
      containers:
      - name: productpage
        image: docker.io/istio/examples-bookinfo-productpage-v1:1.18.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        securityContext:
          runAsUser: 1000
      volumes:
      - name: tmp
        emptyDir: {}
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-productpage
  namespace: demo
---
apiVersion: v1
kind: Service
metadata:
  name: details
  namespace: demo
  labels:
    app: details
    service: details
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: details
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: details-v1
  namespace: demo
  labels:
    app: details
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: details
      version: v1
  template:
    metadata:
      labels:
        app: details
        version: v1
    spec:
      serviceAccountName: bookinfo-details
      containers:
      - name: details
        image: docker.io/istio/examples-bookinfo-details-v1:1.18.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
        securityContext:
          runAsUser: 1000
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-details
  namespace: demo
---
apiVersion: v1
kind: Service
metadata:
  name: reviews
  namespace: demo
  labels:
    app: reviews
    service: reviews
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: reviews
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v1
  namespace: demo
  labels:
    app: reviews
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
      version: v1
  template:
    metadata:
      labels:
        app: reviews
        version: v1
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v1:1.18.0
        imagePullPolicy: IfNotPresent
        env:
        - name: LOG_DIR
          value: "/tmp/logs"
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: wlp-output
          mountPath: /opt/ibm/wlp/output
        securityContext:
          runAsUser: 1000
      volumes:
      - name: wlp-output
        emptyDir: {}
      - name: tmp
        emptyDir: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v2
  namespace: demo
  labels:
    app: reviews
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
      version: v2
  template:
    metadata:
      labels:
        app: reviews
        version: v2
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v2:1.18.0
        imagePullPolicy: IfNotPresent
        env:
        - name: LOG_DIR
          value: "/tmp/logs"
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: wlp-output
          mountPath: /opt/ibm/wlp/output
        securityContext:
          runAsUser: 1000
      volumes:
      - name: wlp-output
        emptyDir: {}
      - name: tmp
        emptyDir: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-v3
  namespace: demo
  labels:
    app: reviews
    version: v3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
      version: v3
  template:
    metadata:
      labels:
        app: reviews
        version: v3
    spec:
      serviceAccountName: bookinfo-reviews
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v3:1.18.0
        imagePullPolicy: IfNotPresent
        env:
        - name: LOG_DIR
          value: "/tmp/logs"
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: wlp-output
          mountPath: /opt/ibm/wlp/output
        securityContext:
          runAsUser: 1000
      volumes:
      - name: wlp-output
        emptyDir: {}
      - name: tmp
        emptyDir: {}
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-reviews
  namespace: demo
---
apiVersion: v1
kind: Service
metadata:
  name: ratings
  namespace: demo
  labels:
    app: ratings
    service: ratings
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: ratings
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ratings-v1
  namespace: demo
  labels:
    app: ratings
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ratings
      version: v1
  template:
    metadata:
      labels:
        app: ratings
        version: v1
    spec:
      serviceAccountName: bookinfo-ratings
      containers:
      - name: ratings
        image: docker.io/istio/examples-bookinfo-ratings-v1:1.18.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
        securityContext:
          runAsUser: 1000
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-ratings
  namespace: demo
```

Deploy:
```bash
kubectl apply -f bookinfo-app.yaml

# Verify pods have sidecar injected (2/2 containers)
kubectl get pods -n demo

# Check services
kubectl get svc -n demo
```

### 5. Create Istio Gateway
Create `bookinfo-gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: demo
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
  name: bookinfo
  namespace: demo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

Apply:
```bash
kubectl apply -f bookinfo-gateway.yaml

# Get ingress gateway external IP
kubectl get svc istio-ingressgateway -n istio-system

export INGRESS_HOST=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

echo "Gateway URL: http://$GATEWAY_URL/productpage"
```

### 6. Configure Traffic Management - Destination Rules
Create `destination-rules.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: productpage
  namespace: demo
spec:
  host: productpage
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
  namespace: demo
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings
  namespace: demo
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: details
  namespace: demo
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
```

Apply:
```bash
kubectl apply -f destination-rules.yaml
```

### 7. Implement Traffic Splitting
Route all traffic to v1:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
```

Apply and test:
```bash
kubectl apply -f reviews-v1.yaml

# Access application multiple times - should only see v1 (no stars)
curl http://$GATEWAY_URL/productpage
```

Split traffic 50/50 between v1 and v3:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```

### 8. Header-Based Routing
Route based on user identity:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: demo
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

Test:
```bash
# Login as 'jason' in the UI to see v2 (black stars)
# Other users see v1 (no stars)
```

### 9. Implement Circuit Breaking
Create circuit breaker policy:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-circuit-breaker
  namespace: demo
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
  subsets:
  - name: v1
    labels:
      version: v1
```

### 10. Configure Request Timeouts
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-timeout
  namespace: demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
    timeout: 0.5s
```

### 11. Implement Retry Logic
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-retry
  namespace: demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx
```

### 12. Fault Injection - Delays
Inject delay for testing:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-delay
  namespace: demo
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 7s
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

### 13. Fault Injection - Aborts
Inject HTTP errors:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-abort
  namespace: demo
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      abort:
        percentage:
          value: 100.0
        httpStatus: 500
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

### 14. Enable mTLS
Apply mesh-wide strict mTLS:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

Verify mTLS:
```bash
kubectl exec -n demo $(kubectl get pod -n demo -l app=productpage -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -- curl -s http://details:9080 -v
```

### 15. Configure Authorization Policies
Deny all by default:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: demo
spec:
  {}
```

Allow specific services:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: productpage-viewer
  namespace: demo
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: details-viewer
  namespace: demo
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/demo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
```

### 16. Configure Telemetry
Enable access logging:

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
  - providers:
    - name: envoy
```

### 17. Install Observability Add-ons
```bash
# Install Kiali, Prometheus, Grafana, Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# Wait for deployments
kubectl rollout status deployment/kiali -n istio-system
kubectl rollout status deployment/prometheus -n istio-system
kubectl rollout status deployment/grafana -n istio-system
```

Access dashboards:
```bash
# Kiali
kubectl port-forward svc/kiali -n istio-system 20001:20001

# Grafana
kubectl port-forward svc/grafana -n istio-system 3000:3000

# Jaeger
kubectl port-forward svc/tracing -n istio-system 16686:80

# Prometheus
kubectl port-forward svc/prometheus -n istio-system 9090:9090
```

### 18. Generate Traffic for Observability
```bash
# Generate continuous traffic
for i in {1..1000}; do
  curl -s http://$GATEWAY_URL/productpage > /dev/null
  echo "Request $i"
  sleep 0.5
done
```

### 19. Analyze with istioctl
```bash
# Analyze mesh configuration
istioctl analyze -n demo

# Validate proxy configuration
istioctl proxy-status

# Get proxy config for specific pod
POD=$(kubectl get pod -n demo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config cluster $POD -n demo

# View listeners
istioctl proxy-config listener $POD -n demo

# View routes
istioctl proxy-config route $POD -n demo

# View endpoints
istioctl proxy-config endpoint $POD -n demo
```

### 20. Cleanup
```bash
# Delete sample application
kubectl delete -f bookinfo-app.yaml
kubectl delete -f bookinfo-gateway.yaml
kubectl delete -f destination-rules.yaml

# Delete namespace
kubectl delete namespace demo

# Uninstall Istio
istioctl uninstall --purge -y

# Delete Istio namespace
kubectl delete namespace istio-system
```

## Expected Results
- Istio control plane running in istio-system
- Automatic sidecar injection in labeled namespaces
- Traffic management with VirtualService and DestinationRule
- mTLS enabled for service-to-service communication
- Circuit breaking and fault injection working
- Authorization policies enforcing access control
- Observability dashboards showing metrics and traces
- Distributed tracing with Jaeger

## Key Takeaways
- **Istio** provides service mesh capabilities for Kubernetes
- **Sidecar proxy (Envoy)** intercepts all network traffic
- **VirtualService** configures routing rules
- **DestinationRule** defines traffic policies for subsets
- **Gateway** manages ingress/egress traffic
- **mTLS** encrypts service-to-service communication
- **Circuit breaking** prevents cascading failures
- **Fault injection** enables chaos engineering
- **Authorization policies** control service access
- **Observability** includes metrics, logs, and traces

## Istio Components

| Component | Purpose |
|-----------|---------|
| Istiod | Control plane (config, discovery, certs) |
| Envoy Proxy | Sidecar data plane |
| Ingress Gateway | Entry point to mesh |
| Egress Gateway | Exit point from mesh |

## Traffic Management Resources

| Resource | Purpose |
|----------|---------|
| VirtualService | Routing rules |
| DestinationRule | Traffic policies |
| Gateway | Ingress/egress |
| ServiceEntry | External services |
| Sidecar | Proxy configuration |

## Troubleshooting
- **Sidecar not injected**: Check namespace label
- **503 errors**: Verify DestinationRule subsets match labels
- **mTLS issues**: Check PeerAuthentication policies
- **Gateway not accessible**: Verify LoadBalancer service
- **Use `istioctl analyze`**: Detect configuration issues

---


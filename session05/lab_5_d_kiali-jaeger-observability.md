# Lab 5d: Kiali and Jaeger Observability
<img width="1509" height="814" alt="ZIMAGE" src="https://github.com/user-attachments/assets/8119010c-c9fd-4628-903f-99438312dd3f" />

## Objective
Deploy and configure Kiali for service mesh visualization and Jaeger for distributed tracing on AKS with Istio. Master observability tools to monitor microservices traffic, analyze service topology, and trace request flows across the mesh.

---

## Lab Parameters

Set these variables at the start:

```bash
# Azure Resources
RESOURCE_GROUP="rg-aks-istio-observability"
LOCATION="australiaeast"
CLUSTER_NAME="aks-istio-obs-cluster"
NODE_COUNT=3
NODE_SIZE="Standard_D4s_v3"
K8S_VERSION="1.28"

# Istio Configuration
ISTIO_VERSION="1.20"
ISTIO_NS="istio-system"
APP_NS="observability-demo"

# Application Configuration
FRONTEND_IMAGE="gcr.io/google-samples/microservices-demo/frontend:v0.8.0"
PRODUCTCATALOG_IMAGE="gcr.io/google-samples/microservices-demo/productcatalogservice:v0.8.0"
RECOMMENDATION_IMAGE="gcr.io/google-samples/microservices-demo/recommendationservice:v0.8.0"

# Observability Tools
SAMPLING_RATE="100.0"
```

---

## Step 1: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \      # `Resource group name`
  --location $LOCATION          # `Azure region (Sydney, Australia)`
```

---

## Step 2: Create AKS Cluster

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --name $CLUSTER_NAME \                   # `Cluster name`
  --location $LOCATION \                   # `Region`
  --node-count $NODE_COUNT \               # `Number of nodes`
  --node-vm-size $NODE_SIZE \              # `VM size (4 vCPU, 16GB for Istio)`
  --kubernetes-version $K8S_VERSION \      # `Kubernetes version`
  --enable-managed-identity \              # `Use managed identity`
  --generate-ssh-keys \                    # `Generate SSH keys`
  --network-plugin azure \                 # `Azure CNI networking`
  --no-wait                                # `Don't wait for completion`

az aks wait \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --created                                # `Wait for cluster creation`

az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing                     # `Configure kubectl`
```

---

## Step 3: Install Istio

Download Istio:
```bash
curl -L https://istio.io/downloadIstio | \
  ISTIO_VERSION=$ISTIO_VERSION sh -      # `Download specific version`

export PATH=$PWD/istio-$ISTIO_VERSION/bin:$PATH  # `Add istioctl to PATH`

echo "Istio $ISTIO_VERSION installed"
```

Install Istio control plane:
```bash
istioctl install \
  --set profile=demo \                   # `Demo profile (all features)`
  -y                                     # `Skip confirmation`

echo "Istio control plane installed"
```

---

## Step 4: Create Application Namespace

```bash
kubectl create namespace $APP_NS         # `Create namespace`

kubectl label namespace $APP_NS \
  istio-injection=enabled                # `Enable automatic sidecar injection`

kubectl get namespace $APP_NS --show-labels  # `Verify label`

echo "Namespace $APP_NS created with Istio injection enabled"
```

---

## Step 5: Install Observability Tools

Install Kiali:
```bash
kubectl apply \
  -f samples/addons/kiali.yaml           # `Kiali service mesh dashboard`

kubectl rollout status \
  deployment/kiali \
  --namespace $ISTIO_NS \
  --timeout=300s                         # `Wait for Kiali to be ready`
```

Install Jaeger:
```bash
kubectl apply \
  -f samples/addons/jaeger.yaml          # `Jaeger distributed tracing`

kubectl rollout status \
  deployment/jaeger \
  --namespace $ISTIO_NS \
  --timeout=300s                         # `Wait for Jaeger to be ready`
```

Install Prometheus:
```bash
kubectl apply \
  -f samples/addons/prometheus.yaml      # `Metrics collection`

kubectl rollout status \
  deployment/prometheus \
  --namespace $ISTIO_NS \
  --timeout=300s                         # `Wait for Prometheus to be ready`
```

Install Grafana:
```bash
kubectl apply \
  -f samples/addons/grafana.yaml         # `Metrics visualization`

kubectl rollout status \
  deployment/grafana \
  --namespace $ISTIO_NS \
  --timeout=300s                         # `Wait for Grafana to be ready`
```

Verify all addons:
```bash
kubectl get pods --namespace $ISTIO_NS | grep -E "kiali|jaeger|prometheus|grafana"  # `Check addon pods`

echo "All observability tools installed"
```

---

## Step 6: Deploy Multi-Service Application

Deploy frontend service:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: frontend
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
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        version: v1
    spec:
      containers:
      - name: frontend
        image: $FRONTEND_IMAGE
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: "productcatalog:3550"
        - name: RECOMMENDATION_SERVICE_ADDR
          value: "recommendation:8080"
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
EOF
```

Deploy product catalog service:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: productcatalog
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
        image: $PRODUCTCATALOG_IMAGE
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
EOF
```

Deploy recommendation service:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: recommendation
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
        image: $RECOMMENDATION_IMAGE
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
EOF
```

Verify deployment:
```bash
kubectl get pods --namespace $APP_NS  # `Check pods (should show 2/2 containers)`

kubectl wait \
  --for=condition=ready pod \
  --all \
  --namespace $APP_NS \
  --timeout=300s                       # `Wait for all pods to be ready`

echo "Application deployed with $APP_NS namespace"
```

---

## Step 7: Create Istio Gateway

Deploy Gateway and VirtualService:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: frontend-gateway
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
EOF

kubectl wait \
  --for=jsonpath='{.status.loadBalancer.ingress[0].ip}' \
  svc/istio-ingressgateway \
  --namespace $ISTIO_NS \
  --timeout=300s                       # `Wait for external IP`

GATEWAY_URL=$(kubectl get svc istio-ingressgateway \
  --namespace $ISTIO_NS \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')  # `Get ingress IP`

echo "Gateway URL captured: $GATEWAY_URL"
echo "Application URL: http://$GATEWAY_URL"
```

---

## Step 8: Generate Traffic for Observability

Generate continuous traffic:
```bash
while true; do \
  curl -s "http://$GATEWAY_URL" > /dev/null; \
  sleep 1; \
done &                                 # `Run in background`

TRAFFIC_PID=$!                         # `Save process ID`
echo "Traffic generator running (PID: $TRAFFIC_PID)"
```

---

## Step 9: Access Kiali Dashboard

Open Kiali using istioctl:
```bash
istioctl dashboard kiali --address 0.0.0.0  # `Open Kiali (http://localhost:20001)`
```

Alternatively, use port-forwarding:
```bash
kubectl port-forward \
  --namespace $ISTIO_NS \
  --address 0.0.0.0 \
  svc/kiali 20001:20001 &              # `Port-forward to Kiali`

echo "Kiali dashboard: http://localhost:20001"
```

---

## Step 10: Explore Kiali Service Graph

In Kiali dashboard:

**1. Navigate to Graph View:**
- Click **Graph** in left sidebar
- Select namespace: `$APP_NS`
- Choose display: **Versioned app graph**

**2. Enable Visualizations:**
- **Traffic Animation**: Shows real-time request flow
- **Response Time**: Displays latency on edges
- **Security**: Shows mTLS lock icons
- **Service Nodes**: Displays service topology

**3. Analyze Topology:**
- Observe: frontend → productcatalog
- Observe: frontend → recommendation → productcatalog
- Check: All connections show mTLS (lock icons)
- Monitor: Traffic percentages and response times

---


## Step 11: Access Jaeger Dashboard

Open Jaeger using istioctl:
```bash
istioctl dashboard jaeger --address 0.0.0.0  # `Open Jaeger (http://localhost:16686)`
```

Alternatively, use port-forwarding:
```bash
kubectl port-forward \
  --namespace $ISTIO_NS \
  --address 0.0.0.0 \
  svc/tracing 16686:80 &               # `Port-forward to Jaeger`

echo "Jaeger UI: http://localhost:16686"
```

---

## Step 12: Search and Analyze Traces

In Jaeger UI:

**Search for Traces:**
1. **Service**: Select `frontend.$APP_NS`
2. **Operation**: Select `all` or specific operation
3. **Lookback**: `1h` (last hour)
4. Click **Find Traces**

**Analyze Trace Details:**
- Click on any trace to expand
- **Span Timeline**: Visual representation of service calls
- **Duration**: Total request duration
- **Spans**: Individual service operations
- **Tags**: HTTP method, status code, etc.

---

## Step 13: Examine Span Details

In a trace view:

**Span Information:**
- **Operation Name**: HTTP GET, gRPC call, etc.
- **Duration**: Time taken for this operation
- **Tags**: Request metadata (http.method, http.status_code, etc.)
- **Logs**: Events within span
- **Process**: Service and version info

**Identify Bottlenecks:**
- Look for spans with high duration
- Check for sequential vs parallel calls
- Analyze error spans (status tags)

---

## Step 14: Compare Traces

Select multiple traces for comparison:
1. Check boxes next to 2-3 traces
2. Click **Compare** button
3. Analyze differences:
   - Total duration variance
   - Span count differences
   - Service call patterns
   - Error occurrences

---

## Step 15: Configure Trace Sampling

Adjust Istio sampling rate:
```bash
kubectl apply --namespace $ISTIO_NS -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
data:
  mesh: |
    defaultConfig:
      tracing:
        sampling: $SAMPLING_RATE
        zipkin:
          address: jaeger-collector.$ISTIO_NS:9411

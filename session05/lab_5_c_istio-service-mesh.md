# Lab 5c: Istio Service Mesh Setup

## Objective
Learn to set up and configure Istio service mesh on AKS for advanced traffic management, security, and observability of microservices.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Helm 3 installed
- Basic understanding of Kubernetes services and networking
- At least 8GB free memory for Istio components

---

## Lab Parameters

```bash
# Resource and location settings
RESOURCE_GROUP="rg-aks-istio-lab"
LOCATION="australiaeast"  # Sydney, Australia

# AKS cluster settings
CLUSTER_NAME="aks-istio-lab"
NODE_COUNT=3
NODE_SIZE="Standard_D4s_v3"  # 4 vCPU, 16GB RAM (required for Istio)
K8S_VERSION="1.29"

# Istio settings
ISTIO_VERSION="1.20.2"
ISTIO_PROFILE="demo"  # demo, default, or production
ISTIO_NS="istio-system"

# Application settings
APP_NS="demo"
BOOKINFO_VERSION="1.18.0"
```

---

## Step 1: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \              # `Resource group name`
  --location $LOCATION                  # `Azure region`
```

---

## Step 2: Create AKS Cluster

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \            # `Resource group`
  --name $CLUSTER_NAME \                        # `Cluster name`
  --location $LOCATION \                        # `Azure region`
  --node-count $NODE_COUNT \                    # `Number of nodes`
  --node-vm-size $NODE_SIZE \                   # `VM size (4 vCPU, 16GB for Istio)`
  --kubernetes-version $K8S_VERSION \           # `Kubernetes version`
  --network-plugin azure \                      # `Azure CNI networking`
  --enable-managed-identity \                   # `Use managed identity`
  --generate-ssh-keys                           # `Generate SSH keys`
```

Get credentials:
```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing  # `Merge credentials to kubeconfig`
```

---

## Step 3: Download and Install Istio CLI

Download Istio:
```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=$ISTIO_VERSION sh -  # `Download specific Istio version`
```

Navigate to Istio directory:
```bash
cd istio-$ISTIO_VERSION  # `Change to Istio directory`
```

Add istioctl to PATH:
```bash
export PATH=$PWD/bin:$PATH  # `Add istioctl to PATH`
```

Verify installation:
```bash
istioctl version  # `Show istioctl version`
```

---

## Step 4: Install Istio Control Plane

Install Istio:
```bash
istioctl install \
  --set profile=$ISTIO_PROFILE \  # `Install profile (demo includes all features)`
  -y                               # `Auto-approve installation`
```

Verify Istio installation:
```bash
kubectl get pods --namespace $ISTIO_NS  # `List Istio control plane pods`
kubectl get svc --namespace $ISTIO_NS   # `Show Istio services`
```

---

## Step 5: Enable Automatic Sidecar Injection

Create application namespace:
```bash
kubectl create namespace $APP_NS  # `Create demo namespace`
```

Enable sidecar injection:
```bash
kubectl label namespace $APP_NS istio-injection=enabled  # `Enable automatic sidecar injection`
```

Verify namespace label:
```bash
kubectl get namespace $APP_NS --show-labels  # `Show namespace labels`
```

---

## Step 6: Deploy Bookinfo Sample Application

Deploy the Bookinfo application using Istio's official sample:
```bash
kubectl apply \
  --namespace $APP_NS \                         # `Target namespace`
  -f samples/bookinfo/platform/kube/bookinfo.yaml  # `Bookinfo application manifests`
```

Verify deployment (pods should have 2/2 containers - app + sidecar):
```bash
kubectl get pods --namespace $APP_NS  # `List pods with sidecar proxies`
kubectl get svc --namespace $APP_NS   # `Show application services`
```

---

## Step 7: Create Istio Gateway

Deploy Istio Gateway for external access:
```bash
kubectl apply \
  --namespace $APP_NS \                          # `Target namespace`
  -f samples/bookinfo/networking/bookinfo-gateway.yaml  # `Gateway and VirtualService`
```

Verify Gateway creation:
```bash
kubectl get gateway --namespace $APP_NS         # `Show Istio Gateway`
kubectl get virtualservice --namespace $APP_NS  # `Show VirtualService`
```

---

## Step 8: Get Application URL

Wait for LoadBalancer IP assignment:
```bash
kubectl wait --namespace $ISTIO_NS \
  --for=jsonpath='{.status.loadBalancer.ingress}' \
  svc/istio-ingressgateway \
  --timeout=300s  # `Wait for external IP (max 5 minutes)`
```

Retrieve ingress gateway external IP:
```bash
INGRESS_HOST=$(kubectl get svc istio-ingressgateway \
  --namespace $ISTIO_NS \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')  # `Get external IP`

INGRESS_PORT=$(kubectl get svc istio-ingressgateway \
  --namespace $ISTIO_NS \
  -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')  # `Get HTTP port`

GATEWAY_URL="$INGRESS_HOST:$INGRESS_PORT"  # `Construct gateway URL`

echo "Bookinfo URL: http://$GATEWAY_URL/productpage"
```

Test application:
```bash
curl -s "http://$GATEWAY_URL/productpage" | grep -o "<title>.*</title>"  # `Test application endpoint`
```

---

## Step 9: Configure Traffic Management

Apply destination rules:
```bash
kubectl apply \
  --namespace $APP_NS \                          # `Target namespace`
  -f samples/bookinfo/networking/destination-rule-all.yaml  # `Define service subsets`
```

Route all traffic to v1:
```bash
kubectl apply \
  --namespace $APP_NS \
  -f samples/bookinfo/networking/virtual-service-all-v1.yaml  # `Route to v1 only`
```

Verify traffic routing:
```bash
kubectl get virtualservice --namespace $APP_NS  # `Show virtual services`
```

---

## Step 10: Test Traffic Splitting

50/50 split between reviews v1 and v3:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50        # 50% to v1
    - destination:
        host: reviews
        subset: v3
      weight: 50        # 50% to v3
EOF
```

Test split:
```bash
for i in {1..10}; do curl -s "http://$GATEWAY_URL/productpage" | grep -o "glyphicon-star"; done  # `Test traffic distribution`
```

---

## Step 11: Header-Based Routing

Route based on user header:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason    # If user=jason
    route:
    - destination:
        host: reviews
        subset: v2        # Route to v2
  - route:
    - destination:
        host: reviews
        subset: v1        # Default to v1
EOF
```

---

## Step 12: Configure Resilience Policies

Add circuit breaker to reviews service:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-resilience
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1      # Max TCP connections
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1  # Eject after 1 error
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

Add timeout and retry policies:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
    timeout: 2s          # 2 second timeout
    retries:
      attempts: 3        # Retry 3 times
      perTryTimeout: 2s  # Per-attempt timeout
EOF
```

---

## Step 13: Inject Faults for Testing

Inject delay:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 100     # 100% of requests
        fixedDelay: 7s   # 7 second delay
    route:
    - destination:
        host: ratings
        subset: v1
EOF
```

Inject HTTP abort (50% failure rate):
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        percentage:
          value: 50      # 50% of requests
        httpStatus: 500  # Return HTTP 500
    route:
    - destination:
        host: ratings
        subset: v1
EOF
```

Remove fault injection:
```bash
kubectl delete virtualservice ratings --namespace $APP_NS  # `Clean up fault injection`
```

---

## Step 14: Enable mTLS (Mutual TLS)

Enable strict mTLS for the namespace:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT  # Enforce mTLS for all services
EOF
```

Verify mTLS status:
```bash
istioctl authn tls-check \
  $(kubectl get pod --namespace $APP_NS -l app=productpage -o jsonpath='{.items[0].metadata.name}') \
  productpage.$APP_NS.svc.cluster.local  # `Check TLS status`
```

---

## Step 15: Configure Authorization Policies

Deny all traffic by default:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
spec:
  {}  # Empty spec = deny all
EOF
```

Allow specific traffic to productpage:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: productpage-viewer
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/$ISTIO_NS/sa/istio-ingressgateway-service-account"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/productpage*"]
EOF
```

Allow productpage to call other services:
```bash
kubectl apply --namespace $APP_NS -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: details-reviews-viewer
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/$APP_NS/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
EOF
```

---

## Step 16: Install Observability Tools

Install Prometheus:
```bash
kubectl apply -f samples/addons/prometheus.yaml  # `Metrics collection`
```

Install Grafana:
```bash
kubectl apply -f samples/addons/grafana.yaml     # `Metrics visualization`
```

Install Kiali:
```bash
kubectl apply -f samples/addons/kiali.yaml       # `Service mesh dashboard`
```

Install Jaeger:
```bash
kubectl apply -f samples/addons/jaeger.yaml      # `Distributed tracing`
```

Verify addon deployments:
```bash
kubectl get pods --namespace $ISTIO_NS | grep -E "prometheus|grafana|kiali|jaeger"  # `Check addon pods`
```

---

## Step 17: Access Observability Dashboards

Open Kiali (service mesh topology):
```bash
istioctl dashboard kiali  # `Open Kiali dashboard`
```

Open Grafana (metrics):
```bash
istioctl dashboard grafana  # `Open Grafana dashboard`
```

Open Jaeger (tracing):
```bash
istioctl dashboard jaeger  # `Open Jaeger UI`
```

Alternatively, use port-forwarding:
```bash
kubectl port-forward \
  --namespace $ISTIO_NS \
  svc/kiali 20001:20001 &  # `Access Kiali at http://localhost:20001`

kubectl port-forward \
  --namespace $ISTIO_NS \
  svc/grafana 3000:3000 &  # `Access Grafana at http://localhost:3000`
```

---

## Step 18: Generate Traffic for Observability

Generate continuous traffic:
```bash
while true; do \
  curl -s "http://$GATEWAY_URL/productpage" > /dev/null; \
  sleep 0.5; \
done
```

Watch traffic in Kiali:
- Navigate to Graph tab
- Select "Versioned app graph"
- Enable "Traffic Animation"

---

## Step 19: Analyze Mesh Configuration

Check proxy status:
```bash
istioctl proxy-status  # `Show status of all Envoy proxies`
```

Validate Istio configuration:
```bash
istioctl analyze --namespace $APP_NS  # `Detect configuration issues`
```

View proxy configuration:
```bash
PRODUCTPAGE_POD=$(kubectl get pod --namespace $APP_NS -l app=productpage -o jsonpath='{.items[0].metadata.name}')

istioctl proxy-config routes $PRODUCTPAGE_POD --namespace $APP_NS  # `View route configuration`
istioctl proxy-config clusters $PRODUCTPAGE_POD --namespace $APP_NS  # `View cluster configuration`
```

---

## Expected Results

After completing this lab, you should have:
- ✅ Working AKS cluster with Istio service mesh
- ✅ Bookinfo application deployed with Envoy sidecars (2/2 containers per pod)
- ✅ Istio Gateway exposing application externally
- ✅ Traffic management policies (routing, splits, header-based)
- ✅ Resilience patterns (circuit breakers, timeouts, retries, fault injection)
- ✅ Security features (mTLS encryption, authorization policies)
- ✅ Observability stack (Prometheus, Grafana, Kiali, Jaeger)
- ✅ Service mesh visibility with traffic topology and distributed traces

---

## Key Takeaways

**Istio Service Mesh Benefits**:
- **Traffic Management**: Fine-grained control over traffic routing and load balancing
- **Resilience**: Built-in circuit breakers, timeouts, and retries without code changes
- **Security**: Automatic mTLS encryption and powerful authorization policies
- **Observability**: Comprehensive metrics, logs, and traces for all service-to-service communication

**Istio Architecture**:
- **Control Plane (istiod)**: Configuration management, service discovery, certificate authority
- **Data Plane (Envoy)**: Sidecar proxies intercept all network traffic
- **Ingress Gateway**: Handles external traffic entering the mesh
- **Egress Gateway**: Controls traffic leaving the mesh

---

## Cleanup

**Option 1** - Remove Istio only (keep AKS cluster):
```bash
# Uninstall addons
kubectl delete -f samples/addons/prometheus.yaml  # `Remove Prometheus`
kubectl delete -f samples/addons/grafana.yaml     # `Remove Grafana`
kubectl delete -f samples/addons/kiali.yaml       # `Remove Kiali`
kubectl delete -f samples/addons/jaeger.yaml      # `Remove Jaeger`

# Remove application
kubectl delete namespace $APP_NS  # `Delete application namespace`

# Uninstall Istio
istioctl uninstall --purge -y  # `Remove all Istio components`

# Delete Istio namespace
kubectl delete namespace $ISTIO_NS  # `Remove Istio namespace`
```

**Option 2** - Delete entire resource group:
```bash
az group delete \
  --name $RESOURCE_GROUP \  # `Resource group name`
  --yes \                   # `Skip confirmation`
  --no-wait                 # `Don't wait for completion`
```

---

**Lab Complete!** You've successfully deployed and configured Istio service mesh on AKS with comprehensive traffic management, security, and observability features.

# Lab 3e: Ingress Routing with NGINX Ingress Controller

## Objective
Deploy **NGINX Ingress Controller** on AKS and configure comprehensive Layer 7 routing: path-based routing, host-based routing, URL rewriting, TLS/SSL termination, rate limiting, basic authentication, CORS, and canary deployments.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` and Helm 3 installed
- Sufficient Azure subscription quota for AKS

---

## 1. Set Lab Parameters

```bash
# Azure resource configuration
RESOURCE_GROUP="rg-aks-ingress-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-ingress-demo"

# AKS cluster configuration
NODE_COUNT=3
NODE_VM_SIZE="Standard_DS2_v2"
K8S_VERSION="1.28"

# Ingress configuration
INGRESS_NAMESPACE="ingress-nginx"
INGRESS_REPLICAS=2
APP_NAMESPACE="default"

# Application configuration
FRONTEND_APP="frontend-app"
API_APP="api-app"
ADMIN_APP="admin-app"

# Domain configuration (for testing)
DOMAIN="example.com"
FRONTEND_HOST="frontend.$DOMAIN"
API_HOST="api.$DOMAIN"
ADMIN_HOST="admin.$DOMAIN"

# Display all configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Ingress Namespace: $INGRESS_NAMESPACE"
echo "Ingress Replicas: $INGRESS_REPLICAS"
echo "Domains: $FRONTEND_HOST, $API_HOST, $ADMIN_HOST"
echo "========================="
```

---

## 2. Create Resource Group and AKS Cluster

```bash
# Create resource group
RG_RESULT=$(az group create \
  --name $RESOURCE_GROUP `# Resource group name` \
  --location $LOCATION `# Azure region` \
  --query 'properties.provisioningState' \
  --output tsv)

echo "Resource Group: $RG_RESULT"

# Create AKS cluster (7-12 minutes)
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_VM_SIZE `# Node VM size` \
  --kubernetes-version $K8S_VERSION `# K8s version` \
  --enable-managed-identity `# Use system-assigned managed identity` \
  --generate-ssh-keys `# Auto-generate SSH keys` \
  --query 'provisioningState' \
  --output tsv)

echo "AKS Cluster: $CLUSTER_RESULT"

# Get AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP `# Source resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --overwrite-existing `# Replace existing kubeconfig entry` \
  --output none

# Verify cluster connectivity
echo "Current kubectl context: $(kubectl config current-context)"
kubectl get nodes
```

---

## 3. Install NGINX Ingress Controller

```bash
# Add NGINX Ingress Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Create namespace for ingress controller
kubectl create namespace $INGRESS_NAMESPACE

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace $INGRESS_NAMESPACE `# Ingress namespace` \
  --set controller.replicaCount=$INGRESS_REPLICAS `# Number of replicas` \
  --set controller.nodeSelector."kubernetes\.io/os"=linux `# Linux nodes` \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz `# Health probe path`

# Wait for ingress controller to be ready
kubectl wait --namespace $INGRESS_NAMESPACE \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Verify installation
kubectl get pods -n $INGRESS_NAMESPACE
kubectl get services -n $INGRESS_NAMESPACE

# Get ingress controller LoadBalancer IP (wait for assignment)
kubectl get service ingress-nginx-controller -n $INGRESS_NAMESPACE --watch
```

**Press Ctrl+C once EXTERNAL-IP appears.**

### Capture Ingress IP

```bash
# Get ingress controller public IP
INGRESS_IP=$(kubectl get service ingress-nginx-controller \
  -n $INGRESS_NAMESPACE \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Ingress Controller IP: $INGRESS_IP"
```

---

## 4. Deploy Backend Applications

### Frontend Application

```bash
# Deploy frontend application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $FRONTEND_APP
  namespace: $APP_NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: setup
        image: busybox
        command:
        - sh
        - -c
        - echo "<h1>Frontend Application</h1><p>Version 1.0</p>" > /html/index.html
        volumeMounts:
        - name: html
          mountPath: /html
      volumes:
      - name: html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: $APP_NAMESPACE
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
EOF

# Wait for frontend deployment
kubectl rollout status deployment/$FRONTEND_APP -n $APP_NAMESPACE --timeout=120s
kubectl get pods -n $APP_NAMESPACE -l app=frontend
```

### API Application

```bash
# Deploy API application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $API_APP
  namespace: $APP_NAMESPACE
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: setup
      volumes:
      - name: html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: $APP_NAMESPACE
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
EOF

# Wait for API deployment
kubectl rollout status deployment/$API_APP -n $APP_NAMESPACE --timeout=120s
kubectl get pods -n $APP_NAMESPACE -l app=api
```

### Admin Application

```bash
# Deploy admin application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $ADMIN_APP
  namespace: $APP_NAMESPACE

**App 3 - Admin:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: nginx
      volumes:
      - name: html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
  namespace: $APP_NAMESPACE
spec:
  selector:
    app: admin
  ports:
  - port: 80
    targetPort: 80
EOF

# Wait for admin deployment
kubectl rollout status deployment/$ADMIN_APP -n $APP_NAMESPACE --timeout=120s
kubectl get pods -n $APP_NAMESPACE -l app=admin
```

---

## 5. Create Path-Based Routing Ingress
  selector:
    app: admin
  ports:
  - port: 80
    targetPort: 80
```

Apply all:
```bash
kubectl apply -f frontend-app.yaml
kubectl apply -f api-app.yaml
kubectl apply -f admin-app.yaml
```

### 3. Create Basic Ingress with Path-Based Routing
Create `basic-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

Apply:
```bash
kubectl apply -f basic-ingress.yaml
```

### 4. Test Path-Based Routing
```bash
# Get ingress IP
INGRESS_IP=$(kubectl get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Ingress IP: $INGRESS_IP"

# Test different paths
curl http://$INGRESS_IP/
curl http://$INGRESS_IP/api
curl http://$INGRESS_IP/admin
```

---

## 6. Create Host-Based Routing

```bash
# Create ingress with host-based routing
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
  namespace: $APP_NAMESPACE
spec:
  ingressClassName: nginx
  rules:
  - host: $FRONTEND_HOST
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: $API_HOST
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: $ADMIN_HOST
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
EOF

# Verify host-based ingress
kubectl get ingress host-ingress -n $APP_NAMESPACE
```

### Test Host-Based Routing

```bash
# Test with host headers
curl -H "Host: $FRONTEND_HOST" http://$INGRESS_IP
curl -H "Host: $API_HOST" http://$INGRESS_IP
curl -H "Host: $ADMIN_HOST" http://$INGRESS_IP
```

---

## 7. Configure SSL/TLS with Self-Signed Certificate

```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key `# Private key file` \
  -out tls.crt `# Certificate file` \
  -subj "/CN=*.$DOMAIN/O=example" `# Wildcard certificate`

# Create Kubernetes TLS secret
kubectl create secret tls example-tls \
  --cert=tls.crt `# Certificate file` \
  --key=tls.key `# Private key file` \
  -n $APP_NAMESPACE

# Verify secret creation
kubectl get secret example-tls -n $APP_NAMESPACE

# Create TLS-enabled ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: $APP_NAMESPACE
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - $FRONTEND_HOST
    - $API_HOST
    secretName: example-tls
  rules:
  - host: $FRONTEND_HOST
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: $API_HOST
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
EOF

# Verify TLS ingress
kubectl describe ingress tls-ingress -n $APP_NAMESPACE
```

### Test HTTPS Access

```bash
# Test HTTPS (use -k to accept self-signed certificate)
curl -k -H "Host: $FRONTEND_HOST" https://$INGRESS_IP
curl -k -H "Host: $API_HOST" https://$INGRESS_IP
```

---

## 8. Configure URL Rewriting

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /v1/api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

Test:
```bash
curl http://$INGRESS_IP/v1/api/
# Rewrites to api-service/
```

### 8. Configure Rate Limiting
Create `rate-limit-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limit-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "20"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

### 9. Configure Basic Authentication
Create htpasswd file:

```bash
# Install htpasswd if needed
# apt-get install apache2-utils

htpasswd -c auth admin
# Enter password when prompted

# Create secret
kubectl create secret generic basic-auth \
  --from-file=auth
```

Create `auth-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
spec:
  ingressClassName: nginx
  rules:
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

Test:
```bash
curl -H "Host: admin.example.com" http://$INGRESS_IP
# Returns 401

curl -u admin:password -H "Host: admin.example.com" http://$INGRESS_IP
# Returns 200
```

### 10. Configure CORS
Create `cors-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cors-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

### 11. Configure Custom Error Pages
Create custom error page:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-error-pages
  namespace: default
data:
  404.html: |
    <html>
    <body>
      <h1>404 - Page Not Found</h1>
      <p>Custom error page</p>
    </body>
    </html>
```

### 12. Configure Canary Deployments
Create v2 of frontend:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app-v2
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
      version: v2
  template:
    metadata:
      labels:
        app: frontend
        version: v2
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: setup
        image: busybox
        command:
        - sh
        - -c
        - echo "<h1>Frontend Application V2</h1><p>Canary Version</p>" > /html/index.html
        volumeMounts:
        - name: html
          mountPath: /html
      volumes:
      - name: html
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service-v2
  namespace: default
spec:
  selector:
    app: frontend
    version: v2
  ports:
  - port: 80
    targetPort: 80
```

Create canary ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-canary
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
  - host: frontend.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service-v2
            port:
              number: 80
```

Test:
```bash
for i in {1..10}; do
  curl -H "Host: frontend.example.com" http://$INGRESS_IP
done
# ~20% requests go to V2
```

### 13. Monitor Ingress
```bash
# View ingress resources
kubectl get ingress

# Describe ingress
kubectl describe ingress app-ingress

# View ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50

# Get ingress status
kubectl get ingress app-ingress -o yaml
```

### 14. View Ingress Metrics
```bash
# Access controller metrics
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller-metrics 10254:10254

# In another terminal
curl http://localhost:10254/metrics
```

## Expected Results
- NGINX Ingress Controller deployed and running
- Path-based routing directs traffic to correct services
- Host-based routing works with custom domains
- SSL/TLS termination configured
- Rate limiting enforced
- Basic authentication protects admin endpoints
- Canary deployments route percentage of traffic
- URL rewriting functions correctly

## Cleanup
```bash
kubectl delete ingress --all
kubectl delete deployment frontend-app api-app admin-app frontend-app-v2
kubectl delete service frontend-service api-service admin-service frontend-service-v2
kubectl delete secret example-tls basic-auth

# Uninstall ingress controller
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```

## Key Takeaways
- **Ingress** provides L7 load balancing and routing
- **Path-based routing** routes by URL path
- **Host-based routing** routes by hostname
- **SSL/TLS termination** at ingress controller
- **Annotations** configure NGINX-specific features
- Single LoadBalancer IP for multiple services (cost-efficient)
- **Canary deployments** enable gradual rollouts
- Rate limiting and auth protect backends

## Useful NGINX Ingress Annotations

| Annotation | Purpose |
|------------|---------|
| `rewrite-target` | URL rewriting |
| `ssl-redirect` | Force HTTPS |
| `rate-limit` | Requests per second |
| `auth-type` | Enable authentication |
| `enable-cors` | CORS headers |
| `canary` | Canary deployment |
| `whitelist-source-range` | IP allowlist |

## Troubleshooting
- **404 errors**: Check ingress rules and service names
- **503 errors**: Backend pods not ready
- **TLS not working**: Verify secret exists and matches host
- **Rate limit not working**: Check annotation syntax

---


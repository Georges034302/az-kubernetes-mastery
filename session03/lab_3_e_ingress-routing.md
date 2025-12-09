# Lab 3e: Ingress Routing

## Objective
Deploy NGINX Ingress and configure dynamic path-based routing.

## Prerequisites
- AKS cluster running
- `kubectl` and Helm 3 installed
- DNS name or domain for testing (optional)

## Steps

### 1. Install NGINX Ingress Controller
Add Helm repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Create namespace:
```bash
kubectl create namespace ingress-nginx
```

Install NGINX Ingress:
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

Verify installation:
```bash
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
```

Get LoadBalancer IP:
```bash
kubectl get service ingress-nginx-controller -n ingress-nginx --watch
```

### 2. Deploy Backend Applications
Create multiple backend services for routing demo.

**App 1 - Frontend:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: default
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
  namespace: default
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

**App 2 - API:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
  namespace: default
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
        image: busybox
        command:
        - sh
        - -c
        - echo "<h1>API Service</h1><p>Endpoint: /api/v1</p>" > /html/index.html
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
  name: api-service
  namespace: default
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
```

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
        - echo "<h1>Admin Dashboard</h1><p>Protected Area</p>" > /html/index.html
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
  name: admin-service
  namespace: default
spec:
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

### 5. Create Host-Based Routing
Create `host-based-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
  namespace: default
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
            name: frontend-service
            port:
              number: 80
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

Apply:
```bash
kubectl apply -f host-based-ingress.yaml
```

Test with host headers:
```bash
curl -H "Host: frontend.example.com" http://$INGRESS_IP
curl -H "Host: api.example.com" http://$INGRESS_IP
curl -H "Host: admin.example.com" http://$INGRESS_IP
```

### 6. Configure SSL/TLS with Self-Signed Certificate
Create self-signed certificate:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=*.example.com/O=example"

# Create Kubernetes secret
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key
```

Create `tls-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - frontend.example.com
    - api.example.com
    secretName: example-tls
  rules:
  - host: frontend.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
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

Apply and test:
```bash
kubectl apply -f tls-ingress.yaml

curl -k -H "Host: frontend.example.com" https://$INGRESS_IP
curl -k -H "Host: api.example.com" https://$INGRESS_IP
```

### 7. Configure URL Rewriting
Create `rewrite-ingress.yaml`:

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


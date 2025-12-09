# Lab 3d: Expose with LoadBalancer

## Objective
Expose workloads using a public LoadBalancer.

## Prerequisites
- AKS cluster running
- `kubectl` configured
- Azure subscription with available public IP quota

## Steps

### 1. Deploy Sample Application
Create `web-app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

Apply:
```bash
kubectl apply -f web-app-deployment.yaml
kubectl get pods -l app=web-app
```

### 2. Create LoadBalancer Service
Create `loadbalancer-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Apply and watch:
```bash
kubectl apply -f loadbalancer-service.yaml
kubectl get service web-app-lb --watch
```

**Wait for EXTERNAL-IP** (takes 1-3 minutes).

### 3. Verify LoadBalancer Creation
```bash
# Get external IP
EXTERNAL_IP=$(kubectl get service web-app-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "External IP: $EXTERNAL_IP"

# Test access
curl http://$EXTERNAL_IP
```

Check Azure Load Balancer:
```bash
# Get cluster resource group
CLUSTER_RG=$(az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query "nodeResourceGroup" -o tsv)

# List load balancers
az network lb list \
  --resource-group $CLUSTER_RG \
  -o table

# View backend pools
az network lb address-pool list \
  --resource-group $CLUSTER_RG \
  --lb-name kubernetes \
  -o table
```

### 4. Create LoadBalancer with Static Public IP
Create a static public IP:

```bash
# Create public IP in cluster resource group
PUBLIC_IP_NAME="aks-static-ip"

az network public-ip create \
  --resource-group $CLUSTER_RG \
  --name $PUBLIC_IP_NAME \
  --sku Standard \
  --allocation-method Static

# Get the IP address
STATIC_IP=$(az network public-ip show \
  --resource-group $CLUSTER_RG \
  --name $PUBLIC_IP_NAME \
  --query ipAddress -o tsv)

echo "Static IP: $STATIC_IP"
```

Create service with static IP - `static-lb-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-static-lb
  namespace: default
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "<CLUSTER_RG>"
spec:
  type: LoadBalancer
  loadBalancerIP: "<STATIC_IP>"
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Apply with substitution:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-app-static-lb
  namespace: default
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "$CLUSTER_RG"
spec:
  type: LoadBalancer
  loadBalancerIP: "$STATIC_IP"
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF
```

Verify:
```bash
kubectl get service web-app-static-lb
curl http://$STATIC_IP
```

### 5. Create Internal LoadBalancer
Create `internal-lb-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-internal-lb
  namespace: default
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Apply:
```bash
kubectl apply -f internal-lb-service.yaml
kubectl get service web-app-internal-lb
```

Get internal IP:
```bash
INTERNAL_IP=$(kubectl get service web-app-internal-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Internal IP: $INTERNAL_IP"

# Test from within cluster
kubectl run test-pod --image=busybox --rm -it -- wget -O- http://$INTERNAL_IP
```

### 6. Create LoadBalancer with Custom Subnet
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-custom-subnet-lb
  namespace: default
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "backend-subnet"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### 7. Configure Session Affinity
Create `session-affinity-lb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-sticky-lb
  namespace: default
spec:
  type: LoadBalancer
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Apply and test:
```bash
kubectl apply -f session-affinity-lb.yaml
STICKY_IP=$(kubectl get service web-app-sticky-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Make multiple requests - should hit same pod
for i in {1..10}; do
  curl http://$STICKY_IP
  sleep 1
done
```

### 8. Configure Health Probe
Create `health-probe-lb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-health-lb
  namespace: default
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/health"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-protocol: "http"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-interval: "5"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-num-of-probe: "2"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Update deployment with health endpoint:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-with-health
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app-health
  template:
    metadata:
      labels:
        app: web-app-health
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
```

### 9. Restrict Source IP Ranges
Create `restricted-lb-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-restricted-lb
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  loadBalancerSourceRanges:
  - "203.0.113.0/24"      # Replace with your allowed IP range
  - "198.51.100.0/24"     # Multiple ranges supported
```

Apply:
```bash
kubectl apply -f restricted-lb-service.yaml
```

Test from allowed and blocked IPs.

### 10. Multi-Port LoadBalancer
Create `multi-port-lb.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-multiport-lb
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  - name: https
    protocol: TCP
    port: 443
    targetPort: 443
```

### 11. View LoadBalancer Details
```bash
# View service details
kubectl describe service web-app-lb

# Get all LoadBalancer services
kubectl get services --all-namespaces -o wide | grep LoadBalancer

# View endpoints
kubectl get endpoints web-app-lb

# Check events
kubectl get events --field-selector involvedObject.name=web-app-lb
```

### 12. Monitor LoadBalancer Metrics in Azure
```bash
# Get public IP resource ID
PUBLIC_IP_ID=$(az network public-ip show \
  --resource-group $CLUSTER_RG \
  --name $PUBLIC_IP_NAME \
  --query id -o tsv)

# View metrics (requires Azure CLI with monitor extension)
az monitor metrics list \
  --resource $PUBLIC_IP_ID \
  --metric "ByteCount" \
  --start-time $(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ') \
  --interval PT1M \
  -o table
```

### 13. Configure Idle Timeout
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-timeout-lb
  namespace: default
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-tcp-idle-timeout: "30"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### 14. Test Load Distribution
```bash
# Get LoadBalancer IP
LB_IP=$(kubectl get service web-app-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Make 100 requests and count pod distribution
for i in {1..100}; do
  curl -s http://$LB_IP | grep -o "pod/[^<]*" || echo "unknown"
done | sort | uniq -c
```

### 15. Cleanup and Cost Management
```bash
# Delete services (releases public IPs)
kubectl delete service web-app-lb web-app-static-lb web-app-internal-lb

# Delete static public IP
az network public-ip delete \
  --resource-group $CLUSTER_RG \
  --name $PUBLIC_IP_NAME

# Delete deployment
kubectl delete deployment web-app
```

## Expected Results
- LoadBalancer service automatically provisions Azure Load Balancer
- External IP assigned and accessible from internet
- Traffic distributed across backend pods
- Static IPs remain consistent across service recreations
- Internal LoadBalancers accessible only within VNet
- Health probes monitor backend pod health
- Source IP restrictions enforced at load balancer level

## Cleanup
```bash
kubectl delete service --all
kubectl delete deployment web-app web-app-with-health
```

## Key Takeaways
- **LoadBalancer** service type creates Azure Load Balancer automatically
- **Dynamic IPs** change if service is deleted/recreated
- **Static IPs** must be in cluster's node resource group
- **Internal LoadBalancers** use annotation `azure-load-balancer-internal: "true"`
- **Health probes** can be customized via annotations
- **Session affinity** enables sticky sessions
- **Source ranges** restrict access to specific IPs
- Each LoadBalancer service incurs Azure Load Balancer costs
- Standard SKU supports availability zones

## LoadBalancer vs Ingress

| Feature | LoadBalancer | Ingress |
|---------|--------------|---------|
| Layer | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| Cost | Per service | Shared |
| IP addresses | One per service | One for multiple services |
| SSL/TLS termination | No | Yes |
| Path-based routing | No | Yes |
| Use case | TCP/UDP apps | Web applications |

## Azure LoadBalancer Annotations

| Annotation | Purpose |
|------------|---------|
| `azure-load-balancer-internal` | Create internal LB |
| `azure-load-balancer-resource-group` | Specify RG for static IP |
| `azure-load-balancer-health-probe-request-path` | Custom health check path |
| `azure-load-balancer-tcp-idle-timeout` | Connection timeout (4-30 min) |

## Troubleshooting
- **EXTERNAL-IP stuck on \<pending\>**: Check Azure quota for public IPs
- **Static IP not working**: Verify IP is in node resource group
- **Health probe failing**: Check pod readiness and health endpoint
- **Cannot access LoadBalancer**: Check NSG rules and source IP restrictions

---

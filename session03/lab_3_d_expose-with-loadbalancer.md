# Lab 3d: Expose Workloads with LoadBalancer

## Objective
Expose Kubernetes workloads using **Azure Load Balancer** with public/static/internal IPs. Demonstrate traffic distribution, health probes, session affinity, IP restrictions, and Azure Load Balancer integration with AKS.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure subscription quota for AKS and public IPs

---

## 1. Set Lab Parameters

```bash
# Azure resource configuration
RESOURCE_GROUP="rg-aks-lb-lab"
LOCATION="australiaeast"
CLUSTER_NAME="aks-lb-demo"

# AKS cluster configuration
NODE_COUNT=3
NODE_VM_SIZE="Standard_DS2_v2"
K8S_VERSION="1.28"
NETWORK_PLUGIN="azure"      # Azure CNI for internal LB support

# Application configuration
APP_NAME="web-app"
APP_REPLICAS=3
NAMESPACE="default"

# Static IP configuration
PUBLIC_IP_NAME="aks-static-ip"

# Display all configuration
echo "=== Lab Configuration ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Kubernetes Version: $K8S_VERSION"
echo "App Name: $APP_NAME"
echo "App Replicas: $APP_REPLICAS"
echo "Static IP Name: $PUBLIC_IP_NAME"
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

# Create AKS cluster with Standard Load Balancer (7-12 minutes)
CLUSTER_RESULT=$(az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_VM_SIZE `# Node VM size` \
  --kubernetes-version $K8S_VERSION `# K8s version` \
  --network-plugin $NETWORK_PLUGIN `# Azure CNI (required for internal LB)` \
  --load-balancer-sku standard `# Standard Load Balancer (required)` \
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

# Get node resource group (where Azure creates LB resources)
NODE_RG=$(az aks show \
  --resource-group $RESOURCE_GROUP `# Source resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --query "nodeResourceGroup" `# Auto-generated resource group` \
  --output tsv)

echo "Node Resource Group: $NODE_RG"
```

---

## 3. Deploy Sample Application

```bash
# Create deployment with pod hostname display
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME
  namespace: $NAMESPACE
spec:
  replicas: $APP_REPLICAS
  selector:
    matchLabels:
      app: $APP_NAME
  template:
    metadata:
      labels:
        app: $APP_NAME
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        command: ["/bin/sh"]
        args:
        - "-c"
        - "echo 'Pod: '\$(hostname) > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
EOF

# Wait for deployment to be ready
kubectl rollout status deployment/$APP_NAME \
  -n $NAMESPACE \
  --timeout=120s

# Verify pods are running
kubectl get pods -n $NAMESPACE -l app=$APP_NAME
```

---

## 4. Create LoadBalancer Service (Dynamic Public IP)

```bash
# Create LoadBalancer service (Azure auto-assigns public IP)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-lb
  namespace: $NAMESPACE
spec:
  type: LoadBalancer
  selector:
    app: $APP_NAME
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

# Watch for external IP assignment (takes 1-3 minutes)
kubectl get service ${APP_NAME}-lb -n $NAMESPACE --watch
```

**Press Ctrl+C once EXTERNAL-IP appears.**

### Verify LoadBalancer and Test Access

```bash
# Get external IP
EXTERNAL_IP=$(kubectl get service ${APP_NAME}-lb \
  -n $NAMESPACE \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "External IP: $EXTERNAL_IP"

# Test access (should show pod hostname)
curl http://$EXTERNAL_IP

# Test load distribution across pods
for i in {1..10}; do
  curl -s http://$EXTERNAL_IP
done
```

---

## 5. Verify Azure Load Balancer Integration

```bash
# List load balancers in node resource group
az network lb list \
  --resource-group $NODE_RG `# Auto-generated resource group` \
  --output table

# View backend address pool (contains node IPs)
az network lb address-pool list \
  --resource-group $NODE_RG `# Node resource group` \
  --lb-name kubernetes `# Default LB name in AKS` \
  --output table

# View load balancing rules
az network lb rule list \
  --resource-group $NODE_RG `# Node resource group` \
  --lb-name kubernetes `# Load balancer name` \
  --output table

# View health probes
az network lb probe list \
  --resource-group $NODE_RG `# Node resource group` \
  --lb-name kubernetes `# Load balancer name` \
  --output table
```

---

## 6. Create LoadBalancer with Static Public IP

```bash
# Create static public IP in node resource group (REQUIRED location)
STATIC_IP_RESULT=$(az network public-ip create \
  --resource-group $NODE_RG `# Must be in node resource group` \
  --name $PUBLIC_IP_NAME `# Public IP name` \
  --sku Standard `# Standard SKU required for AKS` \
  --allocation-method Static `# Static allocation` \
  --query 'publicIp.ipAddress' \
  --output tsv)

echo "Static IP: $STATIC_IP_RESULT"

# Store static IP for reuse
STATIC_IP=$STATIC_IP_RESULT

# Create LoadBalancer service with static IP
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-static-lb
  namespace: $NAMESPACE
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "$NODE_RG"
spec:
  type: LoadBalancer
  loadBalancerIP: "$STATIC_IP"
  selector:
    app: $APP_NAME
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

# Wait for service to be ready
kubectl get service ${APP_NAME}-static-lb -n $NAMESPACE --watch
```

**Press Ctrl+C once EXTERNAL-IP matches the static IP.**

### Verify Static IP Assignment

```bash
# Verify service uses the static IP
ASSIGNED_IP=$(kubectl get service ${APP_NAME}-static-lb \
  -n $NAMESPACE \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Assigned IP: $ASSIGNED_IP"
echo "Static IP: $STATIC_IP"

# Test access
curl http://$STATIC_IP
```

---

## 7. Create Internal LoadBalancer

```bash
# Create internal LoadBalancer (private IP from VNet)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-internal-lb
  namespace: $NAMESPACE
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: $APP_NAME
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

# Wait for internal IP assignment
kubectl get service ${APP_NAME}-internal-lb -n $NAMESPACE --watch
```

**Press Ctrl+C once EXTERNAL-IP shows a private IP.**

### Test Internal LoadBalancer

```bash
# Get internal IP
INTERNAL_IP=$(kubectl get service ${APP_NAME}-internal-lb \
  -n $NAMESPACE \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Internal IP: $INTERNAL_IP"

# Test from within cluster using temporary pod
kubectl run test-internal \
  --image=busybox `# Lightweight test image` \
  --rm `# Delete after exit` \
  -it `# Interactive terminal` \
  -- wget -qO- http://$INTERNAL_IP
```

---

## 8. Configure Session Affinity (Sticky Sessions)

```bash
# Create LoadBalancer with client IP affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-sticky-lb
  namespace: $NAMESPACE
spec:
  type: LoadBalancer
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 1 hour sticky session
  selector:
    app: $APP_NAME
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

# Wait for external IP
kubectl get service ${APP_NAME}-sticky-lb -n $NAMESPACE --watch
```

**Press Ctrl+C once EXTERNAL-IP appears.**

### Test Session Affinity

```bash
# Get sticky LB IP
STICKY_IP=$(kubectl get service ${APP_NAME}-sticky-lb \
  -n $NAMESPACE \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Sticky LB IP: $STICKY_IP"

# Make multiple requests - should hit same pod consistently
for i in {1..10}; do
  curl -s http://$STICKY_IP
  sleep 1
done
```

**Expected:** All requests route to the same pod due to ClientIP affinity.

---

## 9. Configure Custom Health Probe

```bash
# Create LoadBalancer with custom health probe configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-health-lb
  namespace: $NAMESPACE
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-interval: "5"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-num-of-probe: "2"
spec:
  type: LoadBalancer
  selector:
    app: $APP_NAME
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

# View service with health probe annotations
kubectl describe service ${APP_NAME}-health-lb -n $NAMESPACE | grep -A 5 "Annotations"
```

---

## 10. Restrict Source IP Ranges

```bash
# Get your current public IP for testing
MY_IP=$(curl -s https://api.ipify.org)
echo "Your current IP: $MY_IP"

# Create LoadBalancer with IP restrictions
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-restricted-lb
  namespace: $NAMESPACE
spec:
  type: LoadBalancer
  selector:
    app: $APP_NAME
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  loadBalancerSourceRanges:
  - "$MY_IP/32"           # Only allow your current IP
  - "203.0.113.0/24"      # Example additional range
EOF

# Test access (should work from your IP)
RESTRICTED_IP=$(kubectl get service ${APP_NAME}-restricted-lb \
  -n $NAMESPACE \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Restricted LB IP: $RESTRICTED_IP"
curl http://$RESTRICTED_IP
```

---

## 11. Multi-Port LoadBalancer

```bash
# Create LoadBalancer exposing multiple ports
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-multiport-lb
  namespace: $NAMESPACE
spec:
  type: LoadBalancer
  selector:
    app: $APP_NAME
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  - name: https
    protocol: TCP
    port: 443
    targetPort: 80  # Redirect to port 80 for demo
EOF

# View service with multiple ports
kubectl get service ${APP_NAME}-multiport-lb -n $NAMESPACE
kubectl describe service ${APP_NAME}-multiport-lb -n $NAMESPACE | grep -A 3 "Port:"
```

---

## 12. View LoadBalancer Details and Diagnostics

```bash
# View detailed service information
kubectl describe service ${APP_NAME}-lb -n $NAMESPACE

# Get all LoadBalancer services
kubectl get services --all-namespaces -o wide | grep LoadBalancer

# View endpoints (backend pods)
kubectl get endpoints ${APP_NAME}-lb -n $NAMESPACE

# Check service events
kubectl get events \
  --field-selector involvedObject.name=${APP_NAME}-lb \
  -n $NAMESPACE

# View service in YAML format
kubectl get service ${APP_NAME}-lb -n $NAMESPACE -o yaml
```

---

## 13. Monitor LoadBalancer Metrics in Azure

```bash
# Get public IP resource ID for the static IP
PUBLIC_IP_ID=$(az network public-ip show \
  --resource-group $NODE_RG `# Node resource group` \
  --name $PUBLIC_IP_NAME `# Static IP name` \
  --query id `# Resource ID for monitoring` \
  --output tsv)

echo "Public IP Resource ID: $PUBLIC_IP_ID"

# List available metrics for the public IP
az monitor metrics list-definitions \
  --resource $PUBLIC_IP_ID `# Target resource` \
  --output table

# View byte count metrics (last hour)
az monitor metrics list \
  --resource $PUBLIC_IP_ID `# Target resource` \
  --metric "ByteCount" `# Data transferred metric` \
  --interval PT1M `# 1-minute intervals` \
  --output table
```

---

## 14. Configure Idle Timeout

```bash
# Create LoadBalancer with custom TCP idle timeout
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-timeout-lb
  namespace: $NAMESPACE
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-tcp-idle-timeout: "30"
spec:
  type: LoadBalancer
  selector:
    app: $APP_NAME
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

# Verify annotation
kubectl describe service ${APP_NAME}-timeout-lb -n $NAMESPACE | grep -i timeout
```

---

## 15. Test Load Distribution

```bash
# Get LoadBalancer IP
LB_IP=$(kubectl get service ${APP_NAME}-lb \
  -n $NAMESPACE \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Testing load distribution across $APP_REPLICAS pods..."

# Make 50 requests and count pod distribution
for i in {1..50}; do
  curl -s http://$LB_IP
done | sort | uniq -c

echo "Load distribution complete"
```

**Expected:** Requests distributed across all 3 replicas.

---

## Expected Results

✅ LoadBalancer service automatically provisions Azure Load Balancer  
✅ External IP assigned and accessible from internet  
✅ Traffic distributed evenly across backend pods  
✅ Static IPs remain consistent across service recreations  
✅ Internal LoadBalancers accessible only within VNet  
✅ Session affinity routes clients to same pod consistently  
✅ Source IP restrictions enforced at load balancer level  
✅ Health probes monitor backend pod availability  
✅ Multiple ports supported on single LoadBalancer  

---

## Cleanup

### Option 1: Delete Kubernetes Resources Only
```bash
# Delete all LoadBalancer services (releases public IPs)
kubectl delete service ${APP_NAME}-lb ${APP_NAME}-static-lb \
  ${APP_NAME}-internal-lb ${APP_NAME}-sticky-lb \
  ${APP_NAME}-health-lb ${APP_NAME}-restricted-lb \
  ${APP_NAME}-multiport-lb ${APP_NAME}-timeout-lb \
  -n $NAMESPACE

# Delete deployment
kubectl delete deployment $APP_NAME -n $NAMESPACE

# Delete static public IP
az network public-ip delete \
  --resource-group $NODE_RG `# Node resource group` \
  --name $PUBLIC_IP_NAME `# Static IP name`
```

### Option 2: Delete Entire Resource Group (Complete Cleanup)
```bash
# Delete entire resource group (removes cluster, LB, and all resources)
az group delete \
  --name $RESOURCE_GROUP `# Target resource group` \
  --yes `# Skip confirmation` \
  --no-wait `# Run asynchronously`
```

---

## Key Takeaways

- **LoadBalancer** service type automatically provisions Azure Standard Load Balancer
- **Dynamic IPs** change if service is deleted/recreated; use static IPs for persistence
- **Static IPs** must be created in the **node resource group** (not the cluster RG)
- **Internal LoadBalancers** use annotation `service.beta.kubernetes.io/azure-load-balancer-internal: "true"`
- **Azure CNI** is required for internal LoadBalancers with custom subnets
- **Health probes** are automatically configured; customizable via annotations
- **Session affinity** (ClientIP) enables sticky sessions for stateful applications
- **loadBalancerSourceRanges** restricts access to specific IP ranges
- Each LoadBalancer service incurs **Azure Load Balancer costs** (pay per service)
- **Standard Load Balancer SKU** is required for AKS (Basic is deprecated)
- LoadBalancers support **availability zones** for high availability

---

## How This Connects to Application Exposure

This lab demonstrates **Layer 4 (TCP/UDP) load balancing** in Azure:

### LoadBalancer Service Flow:

```
┌─────────────────────────────────────────┐
│  Internet / VNet                        │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  Azure Standard Load Balancer           │
│  (Auto-provisioned by AKS)              │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │ Frontend IP (Public/Internal)    │  │
│  ├──────────────────────────────────┤  │
│  │ Load Balancing Rules             │  │
│  ├──────────────────────────────────┤  │
│  │ Health Probes                    │  │
│  ├──────────────────────────────────┤  │
│  │ Backend Pool (Node IPs)          │  │
│  └──────────────────────────────────┘  │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  AKS Nodes (kube-proxy)                 │
│                                         │
│  Pod1 ──── Pod2 ──── Pod3              │
└─────────────────────────────────────────┘
```

### LoadBalancer vs Ingress:

| Feature | LoadBalancer | Ingress |
|---------|--------------|---------|
| **OSI Layer** | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| **Cost** | Per service ($$) | Shared across services ($) |
| **IP Addresses** | One per service | One for multiple services |
| **SSL/TLS Termination** | No | Yes |
| **Path-Based Routing** | No | Yes (URL routing) |
| **Host-Based Routing** | No | Yes (domain routing) |
| **Use Case** | Non-HTTP protocols, simple HTTP | Web apps, microservices |
| **Azure Resource** | Azure Load Balancer | Application Gateway / AGIC |

---

## Azure LoadBalancer Annotations

| Annotation | Purpose | Example |
|------------|---------|---------|
| `service.beta.kubernetes.io/azure-load-balancer-internal` | Create internal LB | `"true"` |
| `service.beta.kubernetes.io/azure-load-balancer-resource-group` | Specify RG for static IP | `"MC_myRG_myCluster_eastus"` |
| `service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path` | Custom health check path | `"/health"` |
| `service.beta.kubernetes.io/azure-load-balancer-health-probe-interval` | Probe interval (seconds) | `"5"` |
| `service.beta.kubernetes.io/azure-load-balancer-tcp-idle-timeout` | Connection timeout (minutes) | `"30"` (range: 4-30) |
| `service.beta.kubernetes.io/azure-load-balancer-internal-subnet` | Custom subnet for internal LB | `"backend-subnet"` |

---

## LoadBalancer Best Practices

| Practice | Implementation | Rationale |
|----------|---------------|-----------|
| **Use static IPs for production** | Create public IP in node RG | Prevent IP changes on service updates |
| **Internal LB for backend services** | Use `azure-load-balancer-internal: "true"` | Reduce attack surface |
| **Session affinity for stateful apps** | Set `sessionAffinity: ClientIP` | Maintain user session consistency |
| **IP restrictions for security** | Configure `loadBalancerSourceRanges` | Limit access to known networks |
| **Health probe customization** | Set probe path and interval | Improve failover detection |
| **Monitor LB metrics** | Use Azure Monitor | Track traffic and performance |
| **Limit LoadBalancer services** | Prefer Ingress for HTTP workloads | Reduce costs and IP consumption |

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| **EXTERNAL-IP stuck on \<pending\>** | Azure quota exceeded or LB provisioning failure | Check public IP quota: `az network public-ip list` |
| **Static IP not working** | IP not in node resource group | Verify IP location matches node RG |
| **Health probe failing** | Pod not ready or health endpoint misconfigured | Check `kubectl describe pod` and readiness probes |
| **Cannot access LoadBalancer** | NSG rules blocking traffic | Review NSG rules in node resource group |
| **Source IP restriction not working** | Incorrect CIDR notation | Verify `loadBalancerSourceRanges` format |
| **Session affinity not sticky** | Cache/cookies interfering | Test with curl (no cookies) |
| **High latency** | LoadBalancer timeout too short | Increase `tcp-idle-timeout` annotation |

---

## Additional Resources

- [AKS Load Balancer](https://learn.microsoft.com/azure/aks/load-balancer-standard)
- [Azure Standard Load Balancer](https://learn.microsoft.com/azure/load-balancer/load-balancer-overview)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [AKS Network Concepts](https://learn.microsoft.com/azure/aks/concepts-network)

---

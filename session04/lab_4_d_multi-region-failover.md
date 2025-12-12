# Lab 4d: Multi-Region Failover with Azure Traffic Manager

## Objective
Deploy an application across two AKS clusters in different Australian regions (Australia East and Australia Southeast), configure Azure Traffic Manager for DNS-based failover, and validate automatic failover on primary region failure.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Sufficient Azure quota for two AKS clusters

---

## Lab Parameters

```bash
# Primary region (Australia East) settings
PRIMARY_RG="rg-aks-multiregion-primary"
PRIMARY_LOCATION="australiaeast"  # Sydney, Australia
PRIMARY_CLUSTER="aks-primary-australiaeast"
PRIMARY_REGION_LABEL="Australia-East"

# Secondary region (Australia Southeast) settings
SECONDARY_RG="rg-aks-multiregion-secondary"
SECONDARY_LOCATION="australiasoutheast"  # Melbourne, Australia
SECONDARY_CLUSTER="aks-secondary-australiasoutheast"
SECONDARY_REGION_LABEL="Australia-Southeast"

# AKS cluster settings
NODE_COUNT=2
NODE_SIZE="Standard_D2s_v3"
K8S_VERSION="1.29"

# Application settings
APP_NAMESPACE="default"
APP_NAME="web-app"
APP_REPLICAS=3

# Traffic Manager settings
TRAFFIC_MANAGER_RG="rg-aks-traffic-manager"
TRAFFIC_MANAGER_NAME="tm-aks-multiregion"
DNS_NAME="aks-multiregion-$RANDOM"

# Health probe settings
TM_PROBE_INTERVAL=10
TM_PROBE_TIMEOUT=5
TM_PROBE_MAX_FAILURES=3
TM_TTL=10
```

Display configuration:
```bash
echo "=== Lab 4d: Multi-Region Failover ==="
echo "Primary Region: $PRIMARY_LOCATION ($PRIMARY_REGION_LABEL)"
echo "Primary RG: $PRIMARY_RG"
echo "Primary Cluster: $PRIMARY_CLUSTER"
echo "Secondary Region: $SECONDARY_LOCATION ($SECONDARY_REGION_LABEL)"
echo "Secondary RG: $SECONDARY_RG"
echo "Secondary Cluster: $SECONDARY_CLUSTER"
echo "Node Count: $NODE_COUNT"
echo "Node Size: $NODE_SIZE"
echo "Kubernetes Version: $K8S_VERSION"
echo "Traffic Manager: $TRAFFIC_MANAGER_NAME"
echo "DNS Name: $DNS_NAME"
```

---

## Step 1: Create Resource Groups

Create primary region resource group:
```bash
az group create \
  --name $PRIMARY_RG \              # `Resource group name`
  --location $PRIMARY_LOCATION      # `Azure region`
```

Create secondary region resource group:
```bash
az group create \
  --name $SECONDARY_RG \            # `Resource group name`
  --location $SECONDARY_LOCATION    # `Azure region`
```

Create Traffic Manager resource group:
```bash
az group create \
  --name $TRAFFIC_MANAGER_RG \      # `Resource group name`
  --location $PRIMARY_LOCATION      # `Azure region`
```

---

## Step 2: Create Primary AKS Cluster (Australia East)

```bash
az aks create \
  --resource-group $PRIMARY_RG \                # `Resource group`
  --name $PRIMARY_CLUSTER \                     # `Cluster name`
  --location $PRIMARY_LOCATION \                # `Azure region`
  --node-count $NODE_COUNT \                    # `Number of nodes`
  --node-vm-size $NODE_SIZE \                   # `VM size for nodes`
  --kubernetes-version $K8S_VERSION \           # `Kubernetes version`
  --network-plugin azure \                      # `Azure CNI networking`
  --enable-managed-identity \                   # `Use managed identity`
  --generate-ssh-keys                           # `Generate SSH keys`
```

Get credentials for primary cluster:
```bash
az aks get-credentials \
  --resource-group $PRIMARY_RG \
  --name $PRIMARY_CLUSTER \
  --context primary \
  --overwrite-existing
```

---

## Step 3: Create Secondary AKS Cluster (Australia Southeast)

```bash
az aks create \
  --resource-group $SECONDARY_RG \              # `Resource group`
  --name $SECONDARY_CLUSTER \                   # `Cluster name`
  --location $SECONDARY_LOCATION \              # `Azure region`
  --node-count $NODE_COUNT \                    # `Number of nodes`
  --node-vm-size $NODE_SIZE \                   # `VM size for nodes`
  --kubernetes-version $K8S_VERSION \           # `Kubernetes version`
  --network-plugin azure \                      # `Azure CNI networking`
  --enable-managed-identity \                   # `Use managed identity`
  --generate-ssh-keys                           # `Generate SSH keys`
```

Get credentials for secondary cluster:
```bash
az aks get-credentials \
  --resource-group $SECONDARY_RG \
  --name $SECONDARY_CLUSTER \
  --context secondary \
  --overwrite-existing
```

---

## Step 4: Verify kubectl Contexts

```bash
kubectl config get-contexts
```

---

## Step 5: Deploy Application to Primary Region

Deploy web application with region identifier:
```bash
cat <<EOF | kubectl --context primary apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME
  namespace: $APP_NAMESPACE
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
        - |
          echo "<html><body>" > /html/index.html
          echo "<h1>Multi-Region Failover Demo</h1>" >> /html/index.html
          echo "<p><strong>Region:</strong> $PRIMARY_REGION_LABEL</p>" >> /html/index.html
          echo "<p><strong>Hostname:</strong> \$(hostname)</p>" >> /html/index.html
          echo "<p><strong>Timestamp:</strong> \$(date)</p>" >> /html/index.html
          echo "</body></html>" >> /html/index.html
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
  name: $APP_NAME
  namespace: $APP_NAMESPACE
spec:
  type: LoadBalancer
  selector:
    app: $APP_NAME
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
EOF
```

Capture primary LoadBalancer IP (wait for assignment):
```bash
PRIMARY_LB_IP=$(kubectl --context primary get service $APP_NAME \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Primary LoadBalancer IP: $PRIMARY_LB_IP"
```

---

## Step 6: Deploy Application to Secondary Region

Deploy web application with region identifier:
```bash
cat <<EOF | kubectl --context secondary apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME
  namespace: $APP_NAMESPACE
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
        - |
          echo "<html><body>" > /html/index.html
          echo "<h1>Multi-Region Failover Demo</h1>" >> /html/index.html
          echo "<p><strong>Region:</strong> $SECONDARY_REGION_LABEL</p>" >> /html/index.html
          echo "<p><strong>Hostname:</strong> \$(hostname)</p>" >> /html/index.html
          echo "<p><strong>Timestamp:</strong> \$(date)</p>" >> /html/index.html
          echo "</body></html>" >> /html/index.html
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
  name: $APP_NAME
  namespace: $APP_NAMESPACE
spec:
  type: LoadBalancer
  selector:
    app: $APP_NAME
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
EOF
```

Capture secondary LoadBalancer IP (wait for assignment):
```bash
SECONDARY_LB_IP=$(kubectl --context secondary get service $APP_NAME \
  --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Secondary LoadBalancer IP: $SECONDARY_LB_IP"
```

---

## Step 7: Create Azure Traffic Manager Profile

Create Traffic Manager profile with Priority routing (active-passive):
```bash
az network traffic-manager profile create \
  --name $TRAFFIC_MANAGER_NAME \                # `Traffic Manager profile name`
  --resource-group $TRAFFIC_MANAGER_RG \        # `Resource group`
  --routing-method Priority \                   # `Priority-based routing (active-passive)`
  --unique-dns-name $DNS_NAME \                 # `Unique DNS name prefix`
  --ttl $TM_TTL \                               # `DNS TTL in seconds`
  --protocol HTTP \                             # `Health check protocol`
  --port 80 \                                   # `Health check port`
  --path "/" \                                  # `Health check path`
  --interval $TM_PROBE_INTERVAL \               # `Health probe interval (seconds)`
  --timeout $TM_PROBE_TIMEOUT \                 # `Health probe timeout (seconds)`
  --max-failures $TM_PROBE_MAX_FAILURES         # `Max failures before failover`
```

Capture Traffic Manager FQDN:
```bash
TM_FQDN=$(az network traffic-manager profile show \
  --name $TRAFFIC_MANAGER_NAME \
  --resource-group $TRAFFIC_MANAGER_RG \
  --query dnsConfig.fqdn \
  --output tsv)  # `Get Traffic Manager DNS name`

echo "Traffic Manager FQDN: $TM_FQDN"
echo "Traffic Manager URL: http://$TM_FQDN"
```

---

## Step 8: Get Public IP Resource IDs for Traffic Manager

Get primary cluster node resource group:
```bash
PRIMARY_NODE_RG=$(az aks show \
  --resource-group $PRIMARY_RG \
  --name $PRIMARY_CLUSTER \
  --query nodeResourceGroup \
  --output tsv)  # `Get MC_ resource group containing node resources`

echo "Primary Node RG: $PRIMARY_NODE_RG"
```

Get secondary cluster node resource group:
```bash
SECONDARY_NODE_RG=$(az aks show \
  --resource-group $SECONDARY_RG \
  --name $SECONDARY_CLUSTER \
  --query nodeResourceGroup \
  --output tsv)  # `Get MC_ resource group containing node resources`

echo "Secondary Node RG: $SECONDARY_NODE_RG"
```

Get primary LoadBalancer public IP resource ID:
```bash
PRIMARY_PIP_ID=$(az network public-ip list \
  --resource-group $PRIMARY_NODE_RG \
  --query "[?ipAddress=='$PRIMARY_LB_IP'].id" \
  --output tsv)  # `Find public IP resource by IP address`

echo "Primary Public IP ID: $PRIMARY_PIP_ID"
```

Get secondary LoadBalancer public IP resource ID:
```bash
SECONDARY_PIP_ID=$(az network public-ip list \
  --resource-group $SECONDARY_NODE_RG \
  --query "[?ipAddress=='$SECONDARY_LB_IP'].id" \
  --output tsv)  # `Find public IP resource by IP address`

echo "Secondary Public IP ID: $SECONDARY_PIP_ID"
```

---

## Step 9: Add Endpoints to Traffic Manager

Add primary endpoint (Priority 1 - active):
```bash
az network traffic-manager endpoint create \
  --name primary-endpoint \                     # `Endpoint name`
  --profile-name $TRAFFIC_MANAGER_NAME \        # `Traffic Manager profile`
  --resource-group $TRAFFIC_MANAGER_RG \        # `Resource group`
  --type azureEndpoints \                       # `Azure endpoint type`
  --target-resource-id $PRIMARY_PIP_ID \        # `Public IP resource ID`
  --priority 1 \                                # `Priority 1 (highest)`
  --endpoint-status Enabled                     # `Endpoint enabled`
```

Add secondary endpoint (Priority 2 - standby):
```bash
az network traffic-manager endpoint create \
  --name secondary-endpoint \                   # `Endpoint name`
  --profile-name $TRAFFIC_MANAGER_NAME \        # `Traffic Manager profile`
  --resource-group $TRAFFIC_MANAGER_RG \        # `Resource group`
  --type azureEndpoints \                       # `Azure endpoint type`
  --target-resource-id $SECONDARY_PIP_ID \      # `Public IP resource ID`
  --priority 2 \                                # `Priority 2 (lower)`
  --endpoint-status Enabled                     # `Endpoint enabled`
```

---

## Step 10: Verify Traffic Routing to Primary Region

Test Traffic Manager endpoint multiple times:
```bash
for i in {1..5}; do
  echo "Request $i:"
  curl -s http://$TM_FQDN | grep "Region:"
  echo ""
  sleep 2
done
```

Expected result: All requests route to **Australia-East** (primary region).

---

## Step 11: Check Traffic Manager Endpoint Health

Check primary endpoint status:
```bash
az network traffic-manager endpoint show \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $TRAFFIC_MANAGER_RG \
  --query endpointMonitorStatus \
  --output tsv
```

Check secondary endpoint status:
```bash
az network traffic-manager endpoint show \
  --name secondary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $TRAFFIC_MANAGER_RG \
  --query endpointMonitorStatus \
  --output tsv
```

Expected: Both endpoints show `Online`.

---

## Step 12: Simulate Primary Region Failure

Delete primary region service to trigger failover:
```bash
kubectl --context primary delete service $APP_NAME
```

Wait 30-60 seconds for Traffic Manager health probes to detect failure.

---

## Step 13: Verify Automatic Failover to Secondary Region

Check primary endpoint status (should be Degraded):
```bash
az network traffic-manager endpoint show \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $TRAFFIC_MANAGER_RG \
  --query endpointMonitorStatus \
  --output tsv
```

Test Traffic Manager routing after failover:
```bash
for i in {1..5}; do
  echo "Request $i:"
  curl -s http://$TM_FQDN | grep "Region:"
  echo ""
  sleep 2
done
```

Expected result: All requests now route to **Australia-Southeast** (secondary region).

---

## Step 14: Restore Primary Region

Recreate primary service to restore primary endpoint:
```bash
kubectl --context primary expose deployment $APP_NAME \
  --type=LoadBalancer \  # `Create LoadBalancer service`
  --port=80 \              # `External port`
  --target-port=80         # `Container port`
```

Wait 30-60 seconds for Traffic Manager health probes to detect recovery.

Check primary endpoint status (should return to Online):
```bash
az network traffic-manager endpoint show \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $TRAFFIC_MANAGER_RG \
  --query endpointMonitorStatus \
  --output tsv
```

Verify traffic automatically returns to primary:
```bash
for i in {1..5}; do
  echo "Request $i:"
  curl -s http://$TM_FQDN | grep "Region:"
  echo ""
  sleep 2
done
```

Expected result: Traffic returns to **Australia-East** (primary region).

---

## Step 15: Test Manual Endpoint Control (Optional)

Manually disable primary endpoint:
```bash
az network traffic-manager endpoint update \
  --name primary-endpoint \                     # `Endpoint name`
  --profile-name $TRAFFIC_MANAGER_NAME \        # `Traffic Manager profile`
  --resource-group $TRAFFIC_MANAGER_RG \        # `Resource group`
  --endpoint-status Disabled                    # `Disable endpoint`
```

Verify traffic shifts to secondary:
```bash
curl -s http://$TM_FQDN | grep "Region:"
```

Re-enable primary endpoint:
```bash
az network traffic-manager endpoint update \
  --name primary-endpoint \                     # `Endpoint name`
  --profile-name $TRAFFIC_MANAGER_NAME \        # `Traffic Manager profile`
  --resource-group $TRAFFIC_MANAGER_RG \        # `Resource group`
  --endpoint-status Enabled                     # `Enable endpoint`
```

---

## Expected Results

After completing this lab, you should have:

- ✅ Two AKS clusters running in different Australian regions
- ✅ Traffic Manager profile with Priority routing (active-passive)
- ✅ Primary endpoint with Priority 1 (Australia East)
- ✅ Secondary endpoint with Priority 2 (Australia Southeast)
- ✅ Health probes monitoring endpoint availability every 10 seconds
- ✅ Automatic failover to secondary region on primary failure
- ✅ DNS TTL of 10 seconds for fast failover
- ✅ Automatic failback to primary when restored
- ✅ Manual endpoint control capability

---

## Cleanup

### Option 1: Delete Lab Resources Only

Delete applications from both clusters:
```bash
kubectl --context primary delete deployment $APP_NAME
kubectl --context primary delete service $APP_NAME
kubectl --context secondary delete deployment $APP_NAME
kubectl --context secondary delete service $APP_NAME
```

Delete Traffic Manager profile:
```bash
az network traffic-manager profile delete \
  --name $TRAFFIC_MANAGER_NAME \
  --resource-group $TRAFFIC_MANAGER_RG \
  --yes
```

Delete Traffic Manager resource group:
```bash
az group delete \
  --name $TRAFFIC_MANAGER_RG \
  --yes \
  --no-wait
```

### Option 2: Delete All Resources Including AKS Clusters

Delete all resource groups:
```bash
az group delete \
  --name $PRIMARY_RG \
  --yes \
  --no-wait

az group delete \
  --name $SECONDARY_RG \
  --yes \
  --no-wait

az group delete \
  --name $TRAFFIC_MANAGER_RG \
  --yes \
  --no-wait
```

Clean up kubectl contexts:
```bash
kubectl config delete-context primary
kubectl config delete-context secondary
```

---

## How This Connects to Azure Multi-Region Architecture

### Azure Traffic Manager Architecture

```
                    ┌─────────────────────────────────┐
                    │     Client (User Request)       │
                    │        DNS Query                │
                    └──────────────┬──────────────────┘
                                   │
                                   │ 1. DNS resolution
                                   ▼
                    ┌─────────────────────────────────┐
                    │   Azure Traffic Manager         │
                    │   (DNS-based global load bal.)  │
                    │                                 │
                    │   Routing: Priority             │
                    │   TTL: 10 seconds               │
                    │   Health Probe: HTTP/80 @ /    │
                    └──────────────┬──────────────────┘
                                   │
            ┌──────────────────────┴──────────────────────┐
            │                                             │
            │ 2. Return IP based on priority & health    │
            │                                             │
    ┌───────▼──────────┐                      ┌──────────▼───────┐
    │ Priority 1       │                      │ Priority 2       │
    │ Australia East   │                      │ Australia SE     │
    │ (Primary/Active) │                      │ (Secondary/Stby) │
    └───────┬──────────┘                      └──────────┬───────┘
            │                                             │
            │ 3. HTTP traffic                             │ 3. HTTP traffic
            │    to selected region                       │    (failover only)
            ▼                                             ▼
┌──────────────────────────┐            ┌──────────────────────────┐
│  AKS Cluster (Primary)   │            │  AKS Cluster (Secondary) │
│  australiaeast           │            │  australiasoutheast      │
│                          │            │                          │
│  ┌────────────────────┐  │            │  ┌────────────────────┐  │
│  │ LoadBalancer SVC   │  │            │  │ LoadBalancer SVC   │  │
│  │ Public IP: X.X.X.X │  │            │  │ Public IP: Y.Y.Y.Y │  │
│  └─────────┬──────────┘  │            │  └─────────┬──────────┘  │
│            │              │            │            │              │
│            ▼              │            │            ▼              │
│  ┌────────────────────┐  │            │  ┌────────────────────┐  │
│  │ web-app Deployment │  │            │  │ web-app Deployment │  │
│  │ 3 replicas         │  │            │  │ 3 replicas         │  │
│  │ nginx:alpine       │  │            │  │ nginx:alpine       │  │
│  └────────────────────┘  │            │  └────────────────────┘  │
└──────────────────────────┘            └──────────────────────────┘
            ▲                                             ▲
            │                                             │
            │ Health Probe: HTTP GET /                    │
            │ Interval: 10s, Timeout: 5s, MaxFailures: 3  │
            │                                             │
            └─────────────────┬───────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │ Traffic Manager    │
                    │ Health Monitoring  │
                    │                    │
                    │ Primary: Online    │
                    │ Secondary: Online  │
                    └────────────────────┘
```

### Failover Sequence

**Normal Operation (Primary Online):**
1. Client performs DNS query for `$DNS_NAME.trafficmanager.net`
2. Traffic Manager returns Primary IP (Australia East)
3. Client connects directly to Primary LoadBalancer
4. Health probes continuously check Primary endpoint every 10 seconds

**Primary Failure Detected:**
1. Health probe fails 3 consecutive times (30 seconds total)
2. Traffic Manager marks Primary endpoint as `Degraded`
3. Traffic Manager stops returning Primary IP in DNS responses
4. New DNS queries return Secondary IP (Australia Southeast)
5. Existing connections continue until DNS TTL expires (10 seconds)

**Failover Complete:**
1. All new clients resolve to Secondary IP
2. Traffic flows to Australia Southeast cluster
3. Primary endpoint continues health probing
4. When Primary recovers, Traffic Manager detects `Online` status
5. Traffic automatically fails back to Primary (Priority 1)

### Traffic Manager Routing Methods

| Routing Method | Behavior | Use Case | Lab Usage |
|----------------|----------|----------|-----------|  
| **Priority** | Routes to highest priority available endpoint | Active-passive failover, DR | ✅ Used in this lab |
| **Performance** | Routes to lowest latency endpoint | Active-active, global apps | Alternative approach |
| **Geographic** | Routes based on client geographic location | Data residency, compliance | Regional isolation |
| **Weighted** | Distributes traffic by weight percentage | A/B testing, gradual migration | Traffic splitting |
| **MultiValue** | Returns multiple healthy IPs | Client-side load balancing | Advanced scenarios |
| **Subnet** | Routes based on client IP subnet | Internal routing, VPN users | Enterprise networks |

### Health Probe Configuration

**Parameters Used:**
- **Interval**: 10 seconds - How often Traffic Manager checks endpoint health
- **Timeout**: 5 seconds - Maximum time to wait for health probe response
- **Max Failures**: 3 - Number of consecutive failures before marking endpoint as degraded
- **Protocol**: HTTP - Health check protocol (HTTP, HTTPS, TCP)
- **Port**: 80 - Target port for health check
- **Path**: `/` - HTTP path for health check

**Calculation:**
- **Time to detect failure**: `Interval × Max Failures = 10s × 3 = 30 seconds`
- **Total failover time**: `Detection (30s) + DNS TTL (10s) = ~40 seconds`

### DNS TTL Impact

**TTL = 10 seconds (Lab Setting):**
- ✅ Fast failover (40 seconds total)
- ✅ Quick recovery when primary restored
- ⚠️ Higher DNS query load

**TTL = 300 seconds (Default):**
- ❌ Slow failover (330 seconds = 5.5 minutes)
- ✅ Reduced DNS query load
- ⚠️ Clients may cache old IP during outage

**Recommendation**: Use low TTL (10-60s) for critical applications requiring fast failover. Use higher TTL (300s) for cost optimization when failover speed is less critical.

---

## Key Takeaways

- **Azure Traffic Manager** provides DNS-based global load balancing across Azure regions
- **Priority Routing** enables active-passive failover with automatic health-based switchover
- **Health Probes** continuously monitor endpoint availability (HTTP/HTTPS/TCP)
- **DNS TTL** controls how quickly clients switch to failover endpoint
- **Multi-Region Deployment** increases availability and provides disaster recovery capability
- **Failover Time** = Health Detection + DNS TTL propagation (~40 seconds with 10s TTL)
- **Automatic Failback** returns traffic to primary when health is restored
- **No Application Changes** required - failover is transparent to applications
- **Public IP Resources** must exist in Azure for Traffic Manager endpoint targeting
- **LoadBalancer Services** create public IPs automatically in AKS node resource group

### Traffic Manager vs Azure Front Door

| Feature | Traffic Manager | Azure Front Door |
|---------|----------------|------------------|
| **Layer** | L4 (DNS-based) | L7 (HTTP/HTTPS) |
| **Protocol** | Any (DNS redirect) | HTTP/HTTPS only |
| **Failover Speed** | ~40s (with 10s TTL) | <10s (instant) |
| **Caching** | No | Yes (CDN) |
| **WAF** | No | Yes |
| **SSL Offload** | No | Yes |
| **Cost** | Lower | Higher |
| **Best For** | Multi-protocol apps | Web applications |

**Recommendation**: Use **Traffic Manager** for multi-region failover of any protocol. Use **Azure Front Door** for web applications requiring CDN, WAF, and instant failover.

### Multi-Region Data Considerations

**Stateless Applications (This Lab):**
- ✅ Simple deployment - no data synchronization
- ✅ Fast failover - no data consistency concerns
- ✅ Independent regions - no cross-region dependencies

**Stateful Applications (Production):**
- **Azure Cosmos DB**: Multi-region write, automatic replication
- **Azure SQL Database**: Geo-replication, failover groups
- **Azure Storage**: GRS (Geo-Redundant Storage), RA-GRS
- **Redis Cache**: Geo-replication for session state
- **File Shares**: Azure Files with replication

**Consistency Models:**
- **Strong Consistency**: Synchronous replication (latency penalty)
- **Eventual Consistency**: Async replication (temporary inconsistency)
- **Session Consistency**: Consistent within user session (good balance)

---

## Troubleshooting

### Failover Not Happening

Check endpoint monitor status:
```bash
az network traffic-manager endpoint show \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $TRAFFIC_MANAGER_RG \
  --query endpointMonitorStatus
```

### DNS Not Resolving Correctly

Test DNS resolution:
```bash
nslookup $TM_FQDN
```

### Split-Brain Scenario

Ensure health probes are configured correctly and LoadBalancer services are responding on the health check path.

---



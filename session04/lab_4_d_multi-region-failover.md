# Lab 4d: Multi-Region Failover

## Objective
Deploy workloads across two AKS regions and simulate failover.

## Prerequisites
- Azure subscription with sufficient quota
- Azure CLI and `kubectl` configured
- Terraform or ARM templates (optional)
- Understanding of DNS and traffic management

## Steps

### 1. Deploy AKS Clusters in Two Regions
Create clusters in different Azure regions:

**Region 1 (Primary): East US**
```bash
PRIMARY_RG="aks-multiregion-eastus"
PRIMARY_LOCATION="eastus"
PRIMARY_CLUSTER="aks-primary"

az group create \
  --name $PRIMARY_RG \
  --location $PRIMARY_LOCATION

az aks create \
  --resource-group $PRIMARY_RG \
  --name $PRIMARY_CLUSTER \
  --location $PRIMARY_LOCATION \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --network-plugin azure \
  --generate-ssh-keys
```

**Region 2 (Secondary): West US**
```bash
SECONDARY_RG="aks-multiregion-westus"
SECONDARY_LOCATION="westus"
SECONDARY_CLUSTER="aks-secondary"

az group create \
  --name $SECONDARY_RG \
  --location $SECONDARY_LOCATION

az aks create \
  --resource-group $SECONDARY_RG \
  --name $SECONDARY_CLUSTER \
  --location $SECONDARY_LOCATION \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --network-plugin azure \
  --generate-ssh-keys
```

### 2. Configure kubectl Contexts
```bash
# Get credentials for both clusters
az aks get-credentials \
  --resource-group $PRIMARY_RG \
  --name $PRIMARY_CLUSTER \
  --context aks-primary

az aks get-credentials \
  --resource-group $SECONDARY_RG \
  --name $SECONDARY_CLUSTER \
  --context aks-secondary

# Verify contexts
kubectl config get-contexts

# Rename contexts for clarity
kubectl config rename-context $PRIMARY_CLUSTER primary
kubectl config rename-context $SECONDARY_CLUSTER secondary
```

### 3. Deploy Application to Both Regions
Create deployment manifest:

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
        env:
        - name: REGION
          value: "REPLACE_ME"
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: setup
        image: busybox
        command:
        - sh
        - -c
        - echo "<h1>Multi-Region App</h1><p>Region: $REGION</p><p>Hostname: $(hostname)</p>" > /html/index.html
        env:
        - name: REGION
          value: "REPLACE_ME"
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
  name: web-app
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

Deploy to primary region:
```bash
kubectl config use-context primary

cat web-app.yaml | sed 's/REPLACE_ME/East-US/g' | kubectl apply -f -

# Get LoadBalancer IP
kubectl get service web-app --watch
```

Deploy to secondary region:
```bash
kubectl config use-context secondary

cat web-app.yaml | sed 's/REPLACE_ME/West-US/g' | kubectl apply -f -

# Get LoadBalancer IP
kubectl get service web-app --watch
```

### 4. Configure Azure Traffic Manager
Create Traffic Manager profile:

```bash
TRAFFIC_MANAGER_NAME="aks-multiregion-tm"
DNS_NAME="aksmultiregion$RANDOM"

# Create Traffic Manager profile
az network traffic-manager profile create \
  --name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --routing-method Priority \
  --unique-dns-name $DNS_NAME \
  --ttl 10 \
  --protocol HTTP \
  --port 80 \
  --path "/"

# Get FQDN
TM_FQDN=$(az network traffic-manager profile show \
  --name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --query dnsConfig.fqdn -o tsv)

echo "Traffic Manager URL: http://$TM_FQDN"
```

### 5. Add Endpoints to Traffic Manager
```bash
# Get LoadBalancer public IPs
PRIMARY_LB_IP=$(kubectl --context primary get service web-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
SECONDARY_LB_IP=$(kubectl --context secondary get service web-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Create public IP resources for Traffic Manager endpoints
PRIMARY_PIP_ID=$(az network public-ip list \
  --resource-group $(az aks show --resource-group $PRIMARY_RG --name $PRIMARY_CLUSTER --query nodeResourceGroup -o tsv) \
  --query "[?ipAddress=='$PRIMARY_LB_IP'].id" -o tsv)

SECONDARY_PIP_ID=$(az network public-ip list \
  --resource-group $(az aks show --resource-group $SECONDARY_RG --name $SECONDARY_CLUSTER --query nodeResourceGroup -o tsv) \
  --query "[?ipAddress=='$SECONDARY_LB_IP'].id" -o tsv)

# Add primary endpoint (priority 1)
az network traffic-manager endpoint create \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --type azureEndpoints \
  --target-resource-id $PRIMARY_PIP_ID \
  --priority 1 \
  --endpoint-status Enabled

# Add secondary endpoint (priority 2)
az network traffic-manager endpoint create \
  --name secondary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --type azureEndpoints \
  --target-resource-id $SECONDARY_PIP_ID \
  --priority 2 \
  --endpoint-status Enabled
```

### 6. Verify Traffic Routing
Test the Traffic Manager endpoint:

```bash
# Check which region is serving traffic
for i in {1..10}; do
  curl http://$TM_FQDN
  echo ""
  sleep 2
done
```

Expected: All traffic goes to primary region (East US).

### 7. Configure Health Monitoring
Update Traffic Manager health check:

```bash
az network traffic-manager profile update \
  --name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --interval 10 \
  --timeout 5 \
  --max-failures 3 \
  --path "/" \
  --protocol HTTP
```

### 8. Simulate Primary Region Failure
Scale down primary region deployment to simulate failure:

```bash
kubectl --context primary scale deployment web-app --replicas=0

# Or delete the service to make health check fail
kubectl --context primary delete service web-app
```

Wait for health check to fail (30-60 seconds):
```bash
az network traffic-manager endpoint show \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --query endpointMonitorStatus
```

### 9. Verify Automatic Failover
Test that traffic now goes to secondary region:

```bash
for i in {1..10}; do
  curl http://$TM_FQDN
  echo ""
  sleep 2
done
```

Expected: All traffic now goes to secondary region (West US).

### 10. Implement Geo-Routing
Change Traffic Manager to geographic routing:

```bash
az network traffic-manager profile update \
  --name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --routing-method Geographic

# Assign geographic regions to endpoints
az network traffic-manager endpoint update \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --geo-mapping US

az network traffic-manager endpoint update \
  --name secondary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --geo-mapping WORLD
```

### 11. Deploy Shared Data Layer
Create Azure Cosmos DB for global data replication:

```bash
COSMOS_ACCOUNT="aks-cosmos-$RANDOM"

az cosmosdb create \
  --name $COSMOS_ACCOUNT \
  --resource-group $PRIMARY_RG \
  --locations regionName=$PRIMARY_LOCATION failoverPriority=0 isZoneRedundant=False \
  --locations regionName=$SECONDARY_LOCATION failoverPriority=1 isZoneRedundant=False \
  --enable-multiple-write-locations true \
  --default-consistency-level Session

# Get connection string
COSMOS_CONNECTION=$(az cosmosdb keys list \
  --name $COSMOS_ACCOUNT \
  --resource-group $PRIMARY_RG \
  --type connection-strings \
  --query "connectionStrings[0].connectionString" -o tsv)
```

### 12. Deploy Stateful Application with Global Data
Update application to use Cosmos DB:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stateful-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: stateful-app
  template:
    metadata:
      labels:
        app: stateful-app
    spec:
      containers:
      - name: app
        image: your-registry/app:v1
        env:
        - name: COSMOS_CONNECTION
          valueFrom:
            secretKeyRef:
              name: cosmos-secret
              key: connection-string
        - name: REGION
          value: "REGION_NAME"
```

Create secrets in both regions:
```bash
kubectl --context primary create secret generic cosmos-secret \
  --from-literal=connection-string="$COSMOS_CONNECTION"

kubectl --context secondary create secret generic cosmos-secret \
  --from-literal=connection-string="$COSMOS_CONNECTION"
```

### 13. Implement Active-Active Configuration
Change Traffic Manager to Performance routing for active-active:

```bash
az network traffic-manager profile update \
  --name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --routing-method Performance

# Update both endpoints to equal priority
az network traffic-manager endpoint update \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --priority 1

az network traffic-manager endpoint update \
  --name secondary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --priority 1
```

### 14. Monitor Regional Health
Create monitoring dashboard:

```bash
# Query metrics from both clusters
kubectl --context primary top nodes
kubectl --context primary top pods

kubectl --context secondary top nodes
kubectl --context secondary top pods
```

Create Azure Monitor workbook combining both regions.

### 15. Test Manual Failover
Disable primary endpoint:

```bash
az network traffic-manager endpoint update \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --endpoint-status Disabled
```

Verify traffic shifts to secondary:
```bash
curl http://$TM_FQDN
```

Re-enable primary:
```bash
az network traffic-manager endpoint update \
  --name primary-endpoint \
  --profile-name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG \
  --endpoint-status Enabled
```

### 16. Implement Cross-Region Load Balancing with Azure Front Door
Alternative to Traffic Manager:

```bash
FRONTDOOR_NAME="aks-frontdoor-$RANDOM"

az afd profile create \
  --profile-name $FRONTDOOR_NAME \
  --resource-group $PRIMARY_RG \
  --sku Standard_AzureFrontDoor

# Add origin group
az afd origin-group create \
  --profile-name $FRONTDOOR_NAME \
  --origin-group-name aks-origins \
  --resource-group $PRIMARY_RG \
  --probe-path "/" \
  --probe-protocol Http \
  --probe-request-type GET \
  --probe-interval-in-seconds 30

# Add origins
az afd origin create \
  --profile-name $FRONTDOOR_NAME \
  --origin-group-name aks-origins \
  --origin-name primary-aks \
  --resource-group $PRIMARY_RG \
  --host-name $PRIMARY_LB_IP \
  --origin-host-header $PRIMARY_LB_IP \
  --priority 1 \
  --weight 1000 \
  --enabled-state Enabled \
  --http-port 80

az afd origin create \
  --profile-name $FRONTDOOR_NAME \
  --origin-group-name aks-origins \
  --origin-name secondary-aks \
  --resource-group $PRIMARY_RG \
  --host-name $SECONDARY_LB_IP \
  --origin-host-header $SECONDARY_LB_IP \
  --priority 2 \
  --weight 1000 \
  --enabled-state Enabled \
  --http-port 80
```

### 17. Backup and Disaster Recovery
Configure Velero for cluster backup:

```bash
# Install Velero in both clusters
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts

# Create storage account for backups
az storage account create \
  --name aksbackup$RANDOM \
  --resource-group $PRIMARY_RG \
  --sku Standard_GRS \
  --location $PRIMARY_LOCATION

# Install and configure Velero (both clusters)
# ... (see Velero documentation)
```

### 18. Test Complete Failover Scenario
```bash
# 1. Verify primary is healthy
kubectl --context primary get pods

# 2. Take snapshot of primary state
kubectl --context primary get all --all-namespaces > primary-state.txt

# 3. Simulate region failure
kubectl --context primary delete deployment web-app

# 4. Verify secondary takes over
curl http://$TM_FQDN  # Should show West-US

# 5. Restore primary
kubectl --context primary apply -f web-app.yaml

# 6. Verify both regions healthy
kubectl --context primary get pods
kubectl --context secondary get pods
```

### 19. Monitor Failover with Alerts
Create alert for endpoint status change:

```bash
az monitor metrics alert create \
  --name "traffic-manager-failover" \
  --resource-group $PRIMARY_RG \
  --scopes $(az network traffic-manager profile show --name $TRAFFIC_MANAGER_NAME --resource-group $PRIMARY_RG --query id -o tsv) \
  --condition "avg ProbeAgentCurrentEndpointStateByProfileResourceId == 0" \
  --description "Alert when Traffic Manager endpoint goes down" \
  --evaluation-frequency 1m \
  --window-size 5m \
  --severity 1
```

### 20. Cleanup
```bash
# Delete Traffic Manager
az network traffic-manager profile delete \
  --name $TRAFFIC_MANAGER_NAME \
  --resource-group $PRIMARY_RG

# Delete AKS clusters
az aks delete --resource-group $PRIMARY_RG --name $PRIMARY_CLUSTER --yes --no-wait
az aks delete --resource-group $SECONDARY_RG --name $SECONDARY_CLUSTER --yes --no-wait

# Delete resource groups
az group delete --name $PRIMARY_RG --yes --no-wait
az group delete --name $SECONDARY_RG --yes --no-wait
```

## Expected Results
- Two AKS clusters running in different regions
- Traffic Manager routing traffic based on priority
- Automatic failover to secondary region on primary failure
- Health probes detecting endpoint availability
- Geo-routing distributing traffic by geography
- Shared data layer (Cosmos DB) accessible from both regions
- Monitoring and alerts for regional failures

## Key Takeaways
- **Traffic Manager** provides DNS-based global load balancing
- **Priority routing** enables active-passive failover
- **Performance routing** enables active-active with latency-based routing
- **Geographic routing** routes based on user location
- **Health probes** detect endpoint failures automatically
- **Cosmos DB** provides multi-region data replication
- **Azure Front Door** offers L7 load balancing with CDN
- Multi-region deployments increase availability and reduce latency
- Requires careful data consistency planning

## Traffic Manager Routing Methods

| Method | Use Case |
|--------|----------|
| Priority | Active-passive failover |
| Performance | Lowest latency routing |
| Geographic | Compliance, data residency |
| Weighted | A/B testing, gradual migration |
| MultiValue | Return multiple endpoints |
| Subnet | Route by source IP range |

## Troubleshooting
- **Failover not happening**: Check health probe configuration
- **DNS not resolving**: Wait for TTL expiration (default 60s)
- **Split-brain scenario**: Ensure proper health checks
- **Data inconsistency**: Review Cosmos DB consistency level

---



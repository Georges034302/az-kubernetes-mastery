# Lab 3f: Defender Egress Validation
<img width="800" height="352" alt="ZIMAGE" src="https://github.com/user-attachments/assets/b73c36ae-a611-4f8a-95c7-6a079751b67a" />

## Objective
Validate AKS egress security posture using **Azure Firewall**, **User-Defined Routes (UDR)**, and **Microsoft Defender for Containers** by simulating allowed, blocked, and malicious outbound traffic.

---

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Subscription with permissions to create resources
- Understanding of Azure Firewall, UDR, and Kubernetes egress concepts

---

## 1. Define Environment Variables

```bash
# Resource naming
RESOURCE_GROUP="rg-aks-defender-egress"
LOCATION="australiaeast"
CLUSTER_NAME="aks-defender-egress"
VNET_NAME="vnet-aks-defender"
AKS_SUBNET_NAME="snet-aks"
FW_SUBNET_NAME="AzureFirewallSubnet"
FIREWALL_NAME="afw-aks-egress"
FW_PUBLIC_IP_NAME="pip-afw-egress"
FW_ROUTE_TABLE_NAME="rt-aks-fw"
WORKSPACE_NAME="log-aks-fw"
TEST_NAMESPACE="egress-test"

# Network configuration
VNET_ADDRESS_PREFIX="10.224.0.0/16"
AKS_SUBNET_PREFIX="10.224.0.0/22"
FW_SUBNET_PREFIX="10.224.4.0/26"

# AKS configuration
NODE_COUNT=2
NODE_SIZE="Standard_DS2_v2"
K8S_VERSION="1.29"
```

Display variables:
```bash
echo "RESOURCE_GROUP=$RESOURCE_GROUP"
echo "LOCATION=$LOCATION"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "VNET_NAME=$VNET_NAME"
echo "AKS_SUBNET_NAME=$AKS_SUBNET_NAME"
echo "FW_SUBNET_NAME=$FW_SUBNET_NAME"
echo "FIREWALL_NAME=$FIREWALL_NAME"
echo "FW_PUBLIC_IP_NAME=$FW_PUBLIC_IP_NAME"
echo "FW_ROUTE_TABLE_NAME=$FW_ROUTE_TABLE_NAME"
echo "WORKSPACE_NAME=$WORKSPACE_NAME"
echo "TEST_NAMESPACE=$TEST_NAMESPACE"
echo "VNET_ADDRESS_PREFIX=$VNET_ADDRESS_PREFIX"
echo "AKS_SUBNET_PREFIX=$AKS_SUBNET_PREFIX"
echo "FW_SUBNET_PREFIX=$FW_SUBNET_PREFIX"
echo "NODE_COUNT=$NODE_COUNT"
echo "NODE_SIZE=$NODE_SIZE"
echo "K8S_VERSION=$K8S_VERSION"
```

---

## 2. Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP `# Target resource group` \
  --location $LOCATION `# Azure region`
```

---

## 3. Create Virtual Network with Subnets

```bash
az network vnet create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $VNET_NAME `# Virtual network name` \
  --location $LOCATION `# Azure region` \
  --address-prefixes $VNET_ADDRESS_PREFIX `# VNet CIDR block` \
  --subnet-name $AKS_SUBNET_NAME `# AKS subnet name` \
  --subnet-prefixes $AKS_SUBNET_PREFIX `# AKS subnet CIDR`
```

Retrieve AKS subnet ID:
```bash
AKS_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $AKS_SUBNET_NAME \
  --query id -o tsv)

echo "AKS_SUBNET_ID=$AKS_SUBNET_ID"
```

Create Azure Firewall subnet (must be named `AzureFirewallSubnet`):
```bash
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --vnet-name $VNET_NAME `# Parent virtual network` \
  --name $FW_SUBNET_NAME `# Firewall subnet (required name)` \
  --address-prefix $FW_SUBNET_PREFIX `# Firewall subnet CIDR`
```

---

## 4. Create AKS Cluster with Azure CNI

```bash
az aks create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --location $LOCATION `# Azure region` \
  --kubernetes-version $K8S_VERSION `# Kubernetes version` \
  --node-count $NODE_COUNT `# Initial node count` \
  --node-vm-size $NODE_SIZE `# Node VM size` \
  --network-plugin azure `# Use Azure CNI for advanced networking` \
  --vnet-subnet-id $AKS_SUBNET_ID `# Deploy nodes into existing subnet` \
  --service-cidr 10.240.0.0/16 `# Kubernetes service CIDR` \
  --dns-service-ip 10.240.0.10 `# Kubernetes DNS service IP` \
  --generate-ssh-keys `# Generate SSH keys automatically` \
  --enable-managed-identity `# Use managed identity` \
  --no-wait `# Run asynchronously`
```

Wait for cluster creation:
```bash
az aks wait \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --created \
  --interval 30 \
  --timeout 1800
```

Get cluster credentials:
```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --overwrite-existing `# Overwrite existing context`
```

---

## 5. Enable Microsoft Defender for Containers

Enable Defender for Containers at subscription level:
```bash
az security pricing create \
  --name Containers `# Defender plan name` \
  --tier Standard `# Enable paid tier for advanced protection`
```

Enable Defender profile on AKS cluster:
```bash
az aks update \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $CLUSTER_NAME `# AKS cluster name` \
  --enable-defender `# Enable Defender for Containers profile`
```

---

## 6. Deploy Azure Firewall

Create public IP for firewall:
```bash
az network public-ip create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $FW_PUBLIC_IP_NAME `# Public IP name` \
  --location $LOCATION `# Azure region` \
  --sku Standard `# Standard SKU required for Azure Firewall` \
  --allocation-method Static `# Static IP allocation`
```

Create Azure Firewall:
```bash
az network firewall create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $FIREWALL_NAME `# Firewall name` \
  --location $LOCATION `# Azure region` \
  --enable-dns-proxy true `# Enable DNS proxy for FQDN filtering`
```

Configure firewall IP settings:
```bash
az network firewall ip-config create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --firewall-name $FIREWALL_NAME `# Firewall name` \
  --name fw-ipconfig `# IP configuration name` \
  --public-ip-address $FW_PUBLIC_IP_NAME `# Public IP resource` \
  --vnet-name $VNET_NAME `# Virtual network`
```

Retrieve firewall private IP:
```bash
FIREWALL_PRIVATE_IP=$(az network firewall show \
  --resource-group $RESOURCE_GROUP \
  --name $FIREWALL_NAME \
  --query "ipConfigurations[0].privateIPAddress" -o tsv)

echo "FIREWALL_PRIVATE_IP=$FIREWALL_PRIVATE_IP"
```

---

## 7. Create Route Table and Force Egress Through Firewall

Create route table:
```bash
az network route-table create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $FW_ROUTE_TABLE_NAME `# Route table name` \
  --location $LOCATION `# Azure region`
```

Create default route to send all egress traffic through firewall:
```bash
az network route-table route create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --route-table-name $FW_ROUTE_TABLE_NAME `# Route table name` \
  --name default-fw-route `# Route name` \
  --address-prefix 0.0.0.0/0 `# All internet-bound traffic` \
  --next-hop-type VirtualAppliance `# Route to virtual appliance` \
  --next-hop-ip-address $FIREWALL_PRIVATE_IP `# Firewall private IP`
```

Associate route table with AKS subnet:
```bash
az network vnet subnet update \
  --ids $AKS_SUBNET_ID `# AKS subnet resource ID` \
  --route-table $FW_ROUTE_TABLE_NAME `# Route table to associate`
```

---

## 8. Configure Azure Firewall Network Rules

Create network rule collection for required AKS traffic:
```bash
az network firewall network-rule create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --firewall-name $FIREWALL_NAME `# Firewall name` \
  --collection-name aks-core-net `# Rule collection name` \
  --name allow-aks-api `# Rule name` \
  --protocols TCP `# Protocol type` \
  --source-addresses '*' `# Allow from all sources` \
  --destination-addresses AzureCloud `# Destination service tag` \
  --destination-ports 443 9000 `# HTTPS and AKS tunnel ports` \
  --priority 100 `# Rule priority` \
  --action Allow `# Allow traffic`
```

Allow DNS resolution:
```bash
az network firewall network-rule create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --firewall-name $FIREWALL_NAME `# Firewall name` \
  --collection-name aks-core-net `# Rule collection name` \
  --name allow-dns `# Rule name` \
  --protocols UDP `# Protocol type` \
  --source-addresses '*' `# Allow from all sources` \
  --destination-addresses '*' `# Allow to any DNS server` \
  --destination-ports 53 `# DNS port`
```

Allow NTP time synchronization:
```bash
az network firewall network-rule create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --firewall-name $FIREWALL_NAME `# Firewall name` \
  --collection-name aks-core-net `# Rule collection name` \
  --name allow-ntp `# Rule name` \
  --protocols UDP `# Protocol type` \
  --source-addresses '*' `# Allow from all sources` \
  --destination-addresses '*' `# Allow to any NTP server` \
  --destination-ports 123 `# NTP port`
```

---

## 9. Configure Azure Firewall Application Rules

Allow required FQDNs for AKS operations:
```bash
az network firewall application-rule create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --firewall-name $FIREWALL_NAME `# Firewall name` \
  --collection-name aks-fqdns `# Rule collection name` \
  --name allow-aks-fqdns `# Rule name` \
  --protocols https=443 `# HTTPS protocol` \
  --source-addresses '*' `# Allow from all sources` \
  --target-fqdns '*.azmk8s.io' '*.aks.azure.com' '*.microsoft.com' 'mcr.microsoft.com' '*.data.mcr.microsoft.com' `# Required AKS FQDNs` \
  --priority 100 `# Rule priority` \
  --action Allow `# Allow traffic`
```

Allow container registries:
```bash
az network firewall application-rule create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --firewall-name $FIREWALL_NAME `# Firewall name` \
  --collection-name aks-fqdns `# Rule collection name` \
  --name allow-registries `# Rule name` \
  --protocols https=443 `# HTTPS protocol` \
  --source-addresses '*' `# Allow from all sources` \
  --target-fqdns 'docker.io' '*.docker.io' 'gcr.io' '*.gcr.io' 'quay.io' '*.quay.io' `# Container registries`
```

---

## 10. Create Test Namespace

```bash
kubectl create namespace $TEST_NAMESPACE
```

---

## 11. Deploy Egress Test Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: egress-test
  namespace: $TEST_NAMESPACE
  labels:
    app: egress-test
spec:
  containers:
  - name: test
    image: nicolaka/netshoot
    command:
    - sleep
    - "3600"
EOF
```

Wait for pod to be ready:
```bash
kubectl wait pod/egress-test \
  --namespace $TEST_NAMESPACE \
  --for=condition=Ready \
  --timeout=120s
```

---

## 12. Test Allowed Egress (Should Succeed)

Test Microsoft Container Registry (allowed by firewall):
```bash
kubectl exec egress-test -n $TEST_NAMESPACE -- curl -I https://mcr.microsoft.com
```

Test Microsoft domain (allowed by firewall):
```bash
kubectl exec egress-test -n $TEST_NAMESPACE -- curl -I https://www.microsoft.com
```

Test internal DNS resolution:
```bash
kubectl exec egress-test -n $TEST_NAMESPACE -- nslookup kubernetes.default.svc.cluster.local
```

**Expected Result**: All commands succeed.

---

## 13. Test Blocked Egress (Should Fail)

Test Google domain (not in firewall allow list):
```bash
kubectl exec egress-test -n $TEST_NAMESPACE -- curl -m 10 https://www.google.com
```

Test Example.org (not in firewall allow list):
```bash
kubectl exec egress-test -n $TEST_NAMESPACE -- curl -m 10 https://example.org
```

**Expected Result**: Connection timeout or failure (firewall denial).

---

## 14. Deploy Suspicious Pod for Defender Detection

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: suspicious-pod
  namespace: $TEST_NAMESPACE
spec:
  containers:
  - name: miner
    image: alpine
    command:
    - sh
    - -c
    - |
      apk add --no-cache stress-ng
      stress-ng --cpu 4 --timeout 600
EOF
```

**Expected Defender Detection**:
- High CPU usage pattern
- Suspicious process execution (stress-ng)
- Potential cryptocurrency mining behavior

---

## 15. Simulate Data Exfiltration Attempts

Attempt connection to known Tor exit node IP (should be blocked):
```bash
kubectl exec egress-test -n $TEST_NAMESPACE -- curl -m 10 http://185.220.101.1
```

Attempt netcat connection to suspicious port (should be blocked):
```bash
kubectl exec egress-test -n $TEST_NAMESPACE -- nc -zv 8.8.8.8 4444
```

**Expected Result**: Firewall denies all attempts.

---

## 16. Enable Firewall Threat Intelligence

```bash
az network firewall update \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --name $FIREWALL_NAME `# Firewall name` \
  --threat-intel-mode Alert `# Enable threat intelligence in alert mode`
```

---

## 17. Configure Firewall Logging

Create Log Analytics workspace:
```bash
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP `# Target resource group` \
  --workspace-name $WORKSPACE_NAME `# Workspace name` \
  --location $LOCATION `# Azure region`
```

Retrieve workspace resource ID:
```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME \
  --query id -o tsv)

echo "WORKSPACE_ID=$WORKSPACE_ID"
```

Retrieve firewall resource ID:
```bash
FIREWALL_ID=$(az network firewall show \
  --resource-group $RESOURCE_GROUP \
  --name $FIREWALL_NAME \
  --query id -o tsv)

echo "FIREWALL_ID=$FIREWALL_ID"
```

Enable diagnostic settings for firewall:
```bash
az monitor diagnostic-settings create \
  --resource $FIREWALL_ID `# Firewall resource ID` \
  --name fw-diagnostics `# Diagnostic settings name` \
  --workspace $WORKSPACE_ID `# Log Analytics workspace ID` \
  --logs '[
    {"category":"AzureFirewallApplicationRule","enabled":true},
    {"category":"AzureFirewallNetworkRule","enabled":true},
    {"category":"AzureFirewallDnsProxy","enabled":true}
  ]' `# Enable firewall log categories` \
  --metrics '[{"category":"AllMetrics","enabled":true}]' `# Enable all metrics`
```

---

## 18. Query Denied Traffic Logs

Query application rule denials:
```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID `# Workspace ID` \
  --analytics-query "
    AzureFirewallApplicationRule
    | where Action == 'Deny'
    | project TimeGenerated, SourceIp, Fqdn, Action
    | order by TimeGenerated desc
    | take 20
  " `# Query denied application traffic` \
  -o table
```

Query network rule denials:
```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID `# Workspace ID` \
  --analytics-query "
    AzureFirewallNetworkRule
    | where Action == 'Deny'
    | project TimeGenerated, SourceIp, DestinationIp, DestinationPort, Action
    | order by TimeGenerated desc
    | take 20
  " `# Query denied network traffic` \
  -o table
```

Summarize egress patterns:
```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "
    AzureFirewallApplicationRule
    | summarize Count = count() by Fqdn, Action
    | order by Count desc
    | take 20
  " \
  -o table
```

---

## 19. View Defender for Containers Alerts

List recent security alerts:
```bash
az security alert list \
  --query "[?contains(resourceIdentifiers[0], 'kubernetes')].{
    Name:alertDisplayName,
    Severity:severity,
    Status:status,
    Time:timeGeneratedUtc
  }" `# Filter Kubernetes-related alerts` \
  -o table
```

**Common Defender Alerts**:
- Privileged container detected
- Container with sensitive volume mount detected
- Role binding to cluster-admin role detected
- Suspicious process detected
- Digital currency mining activity detected

---

## 20. Cleanup

**Option 1**: Remove test resources only:
```bash
kubectl delete namespace $TEST_NAMESPACE
```

**Option 2**: Delete entire resource group:
```bash
az group delete \
  --name $RESOURCE_GROUP `# Resource group to delete` \
  --yes `# Skip confirmation` \
  --no-wait `# Run asynchronously`
```

---

## How This Connects to AKS Security

### Egress Control Architecture
```
                    Internet
                       ↑
         Allowed/Denied by Azure Firewall
                       │
        ┌──────────────┴──────────────┐
        │     Azure Firewall          │
        │  • Application Rules (FQDN) │
        │  • Network Rules (IP/Port)  │
        │  • Threat Intelligence      │
        │  • DNS Proxy                │
        └──────────────┬──────────────┘
                       │ UDR (0.0.0.0/0)
        ┌──────────────┴──────────────┐
        │        AKS Subnet            │
        │  • Worker Nodes              │
        │  • Pods                      │
        └──────────────┬──────────────┘
                       │
        ┌──────────────┴──────────────┐
        │  Microsoft Defender          │
        │  • Runtime Monitoring        │
        │  • Threat Detection          │
        │  • Compliance Scanning       │
        └──────────────────────────────┘
                       │
              Log Analytics Workspace
```

### Defense-in-Depth Layers

| Layer | Component | Purpose |
|-------|-----------|---------|
| **Network** | Azure Firewall | Centralized egress filtering by FQDN/IP |
| **Route Control** | User-Defined Routes | Force all traffic through firewall |
| **Application** | Firewall App Rules | Allow/deny based on destination FQDN |
| **Network** | Firewall Net Rules | Allow/deny based on IP/port/protocol |
| **Threat Intel** | Azure Threat Intelligence | Block known malicious IPs/domains |
| **Runtime** | Defender for Containers | Detect suspicious pod behavior |
| **Logging** | Log Analytics | Forensic analysis and alerting |

### Key Security Concepts

**Azure Firewall**:
- Central egress control point for entire AKS cluster
- Layer 7 (application) and Layer 4 (network) filtering
- DNS proxy enables FQDN filtering for non-HTTP traffic
- Threat intelligence feed blocks known malicious destinations
- Stateful inspection of all traffic flows

**User-Defined Routes (UDR)**:
- Override default Azure routing
- Force all outbound (0.0.0.0/0) traffic through firewall
- Applied at subnet level (AKS subnet)
- Cannot be bypassed by pods or nodes

**Defender for Containers**:
- Agentless runtime protection
- Detects anomalous processes (crypto mining, privilege escalation)
- Monitors network connections for data exfiltration
- Analyzes Kubernetes audit logs
- Provides CIS benchmark compliance scanning

**Required AKS Egress**:
- **AzureCloud** service tag (AKS control plane, Azure services)
- **DNS** (UDP 53) for name resolution
- **NTP** (UDP 123) for time synchronization
- **MCR** (mcr.microsoft.com) for system container images
- **AKS FQDNs** (*.azmk8s.io, *.aks.azure.com)

### Common Attack Patterns Detected

| Attack Pattern | Detection Method | Example |
|---------------|------------------|---------|
| **Crypto Mining** | High CPU usage + suspicious processes | stress-ng, xmrig |
| **Data Exfiltration** | Connections to unknown external IPs | Tor exit nodes, unusual destinations |
| **Command & Control** | Connections to known malicious domains | Threat intelligence feed |
| **Privilege Escalation** | Privileged container execution | securityContext.privileged=true |
| **Lateral Movement** | Unexpected pod-to-pod communication | Network policy violations |

---

## Expected Results

✅ **Azure Firewall deployed** with public IP and private IP in VNet  
✅ **Route table** forces all egress through firewall (0.0.0.0/0 UDR)  
✅ **Network rules** allow AKS system traffic (AzureCloud, DNS, NTP)  
✅ **Application rules** allow required FQDNs (MCR, AKS domains, registries)  
✅ **Allowed egress succeeds** (mcr.microsoft.com, microsoft.com)  
✅ **Blocked egress fails** (google.com, example.org timeout)  
✅ **Defender detects** suspicious pod activity (crypto mining simulation)  
✅ **Firewall logs** capture all denied connections in Log Analytics  
✅ **Threat intelligence** enabled to block known malicious IPs  
✅ **Security alerts** generated for policy violations  

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| **Pods can't reach internet** | Route table not associated | Verify UDR on AKS subnet |
| **Allowed sites blocked** | Missing firewall rule | Add FQDN to application rule |
| **Firewall not blocking** | Rule priority issue | Check rule order (lower priority = higher precedence) |
| **Defender not detecting** | Profile not enabled | Run `az aks update --enable-defender` |
| **No logs appearing** | Diagnostic settings missing | Verify firewall diagnostic settings |
| **DNS resolution fails** | DNS proxy disabled | Enable `--enable-dns-proxy true` |
| **AKS nodes unhealthy** | Required FQDNs blocked | Add AKS FQDNs to application rules |

---

## Key Takeaways

- **Azure Firewall** provides centralized egress governance for entire AKS cluster
- **User-Defined Routes (UDR)** enforce traffic flow through firewall (cannot be bypassed)
- **Application rules** control FQDN-based traffic (Layer 7)
- **Network rules** control IP/port-based traffic (Layer 4)
- **Defender for Containers** detects runtime threats (crypto mining, exfiltration, privilege escalation)
- **Threat Intelligence** blocks known malicious IPs and domains automatically
- **Log Analytics** provides forensic evidence and alerting capabilities
- **Required AKS egress** must be explicitly allowed (AzureCloud, DNS, NTP, MCR, AKS FQDNs)
- **Defense-in-depth** combines network controls, runtime protection, and logging

---



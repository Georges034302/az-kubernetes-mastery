# Lab 3f: Defender Egress Validation

## Objective
Simulate threats and validate egress control using Azure Firewall.

## Prerequisites
- AKS cluster running
- Azure Firewall deployed (or Azure Firewall Manager)
- Microsoft Defender for Containers enabled
- `kubectl` and Azure CLI configured

## Steps

### 1. Enable Microsoft Defender for Containers
```bash
# Enable Defender for Containers on subscription
az security pricing create \
  --name Containers \
  --tier Standard

# Enable Defender profile on AKS
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --enable-defender
```

Verify Defender pods:
```bash
kubectl get pods -n kube-system | grep -i defender
```

### 2. Create Azure Firewall (If Not Exists)
```bash
# Set variables
RESOURCE_GROUP="<resource-group>"
LOCATION="eastus"
VNET_NAME="aks-vnet"
FIREWALL_SUBNET_NAME="AzureFirewallSubnet"
FIREWALL_NAME="aks-firewall"
FIREWALL_PUBLIC_IP_NAME="firewall-pip"

# Create firewall subnet (must be named AzureFirewallSubnet)
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $FIREWALL_SUBNET_NAME \
  --address-prefix 10.0.3.0/26

# Create public IP for firewall
az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name $FIREWALL_PUBLIC_IP_NAME \
  --sku Standard \
  --allocation-method Static

# Create Azure Firewall
az network firewall create \
  --resource-group $RESOURCE_GROUP \
  --name $FIREWALL_NAME \
  --location $LOCATION \
  --enable-dns-proxy true

# Configure firewall IP config
az network firewall ip-config create \
  --resource-group $RESOURCE_GROUP \
  --firewall-name $FIREWALL_NAME \
  --name firewall-config \
  --public-ip-address $FIREWALL_PUBLIC_IP_NAME \
  --vnet-name $VNET_NAME

# Get firewall private IP
FIREWALL_PRIVATE_IP=$(az network firewall show \
  --resource-group $RESOURCE_GROUP \
  --name $FIREWALL_NAME \
  --query "ipConfigurations[0].privateIPAddress" -o tsv)

echo "Firewall Private IP: $FIREWALL_PRIVATE_IP"
```

### 3. Create Route Table for AKS
```bash
# Create route table
az network route-table create \
  --resource-group $RESOURCE_GROUP \
  --name aks-firewall-rt

# Add route to send all egress traffic through firewall
az network route-table route create \
  --resource-group $RESOURCE_GROUP \
  --route-table-name aks-firewall-rt \
  --name firewall-route \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FIREWALL_PRIVATE_IP

# Associate with AKS subnet
AKS_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name aks-subnet \
  --query id -o tsv)

az network vnet subnet update \
  --ids $AKS_SUBNET_ID \
  --route-table aks-firewall-rt
```

### 4. Configure Firewall Network Rules
Allow required AKS egress traffic:

```bash
# Create network rule collection
az network firewall network-rule create \
  --resource-group $RESOURCE_GROUP \
  --firewall-name $FIREWALL_NAME \
  --collection-name aks-required \
  --name allow-aks-apis \
  --protocols TCP \
  --source-addresses '*' \
  --destination-addresses AzureCloud \
  --destination-ports 443 9000 \
  --priority 100 \
  --action Allow

# Allow NTP
az network firewall network-rule create \
  --resource-group $RESOURCE_GROUP \
  --firewall-name $FIREWALL_NAME \
  --collection-name aks-required \
  --name allow-ntp \
  --protocols UDP \
  --source-addresses '*' \
  --destination-addresses '*' \
  --destination-ports 123

# Allow DNS
az network firewall network-rule create \
  --resource-group $RESOURCE_GROUP \
  --firewall-name $FIREWALL_NAME \
  --collection-name aks-required \
  --name allow-dns \
  --protocols UDP \
  --source-addresses '*' \
  --destination-addresses '*' \
  --destination-ports 53
```

### 5. Configure Firewall Application Rules
```bash
# Allow required FQDNs for AKS
az network firewall application-rule create \
  --resource-group $RESOURCE_GROUP \
  --firewall-name $FIREWALL_NAME \
  --collection-name aks-fqdns \
  --name allow-aks-fqdns \
  --protocols https=443 \
  --source-addresses '*' \
  --target-fqdns \
    '*.azmk8s.io' \
    '*.microsoft.com' \
    '*.windows.net' \
    'mcr.microsoft.com' \
    '*.data.mcr.microsoft.com' \
  --priority 100 \
  --action Allow

# Allow container registries
az network firewall application-rule create \
  --resource-group $RESOURCE_GROUP \
  --firewall-name $FIREWALL_NAME \
  --collection-name aks-fqdns \
  --name allow-registries \
  --protocols https=443 \
  --source-addresses '*' \
  --target-fqdns \
    'docker.io' \
    '*.docker.io' \
    'gcr.io' \
    '*.gcr.io' \
    'quay.io' \
    '*.quay.io'
```

### 6. Deploy Test Workload
Create `test-egress-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: egress-test
  namespace: default
  labels:
    app: egress-test
spec:
  containers:
  - name: test
    image: nicolaka/netshoot
    command:
    - sleep
    - "3600"
```

Apply:
```bash
kubectl apply -f test-egress-pod.yaml
kubectl wait --for=condition=ready pod/egress-test --timeout=60s
```

### 7. Test Allowed Egress
```bash
# Test allowed destinations
kubectl exec egress-test -- curl -I https://mcr.microsoft.com
kubectl exec egress-test -- curl -I https://www.microsoft.com
kubectl exec egress-test -- nslookup kubernetes.default.svc.cluster.local
```

Expected: All succeed (allowed by firewall rules).

### 8. Test Blocked Egress
```bash
# Test blocked destinations (not in allow list)
kubectl exec egress-test -- curl -m 10 https://www.google.com
kubectl exec egress-test -- curl -m 10 https://www.example.com
```

Expected: Connection timeout (blocked by firewall).

### 9. Simulate Malicious Activity
Create suspicious pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: suspicious-pod
  namespace: default
spec:
  containers:
  - name: bad-actor
    image: alpine
    command:
    - sh
    - -c
    - |
      # Simulate crypto mining
      while true; do
        echo "Mining cryptocurrency..."
        dd if=/dev/zero of=/dev/null &
        sleep 10
      done
    securityContext:
      privileged: true  # Suspicious
      capabilities:
        add:
        - SYS_ADMIN
```

```bash
kubectl apply -f suspicious-pod.yaml
```

### 10. Simulate Data Exfiltration Attempt
```bash
# Try to connect to suspicious external IPs
kubectl exec egress-test -- curl -m 10 http://185.220.101.1
kubectl exec egress-test -- curl -m 10 http://tor-exit-node.example.com

# Try to use alternate ports
kubectl exec egress-test -- nc -zv 8.8.8.8 8080
kubectl exec egress-test -- nc -zv suspicious-ip.com 4444
```

### 11. View Defender Alerts
```bash
# Check Defender recommendations in Azure Portal
# Security Center > Recommendations > Containers

# View alerts via CLI
az security alert list \
  --query "[?contains(resourceIdentifiers[0], 'kubernetes')].{Name:alertDisplayName, Severity:severity, Time:timeGeneratedUtc}" \
  -o table
```

### 12. Monitor Firewall Logs
Enable diagnostic settings:

```bash
# Create Log Analytics workspace
WORKSPACE_NAME="aks-firewall-logs"
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME

WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME \
  --query id -o tsv)

# Enable diagnostic settings for firewall
az monitor diagnostic-settings create \
  --resource $(az network firewall show --resource-group $RESOURCE_GROUP --name $FIREWALL_NAME --query id -o tsv) \
  --name firewall-diagnostics \
  --workspace $WORKSPACE_ID \
  --logs '[
    {
      "category": "AzureFirewallApplicationRule",
      "enabled": true
    },
    {
      "category": "AzureFirewallNetworkRule",
      "enabled": true
    }
  ]' \
  --metrics '[
    {
      "category": "AllMetrics",
      "enabled": true
    }
  ]'
```

Query firewall logs:
```bash
# View blocked connections
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "
    AzureFirewallApplicationRule
    | where Action == 'Deny'
    | project TimeGenerated, SourceIp, Fqdn, Action
    | order by TimeGenerated desc
    | take 20
  " \
  -o table
```

### 13. Create Firewall Deny Rule for Testing
```bash
# Create explicit deny rule for testing
az network firewall application-rule create \
  --resource-group $RESOURCE_GROUP \
  --firewall-name $FIREWALL_NAME \
  --collection-name block-social-media \
  --name block-facebook \
  --protocols https=443 \
  --source-addresses '*' \
  --target-fqdns \
    'facebook.com' \
    '*.facebook.com' \
    'twitter.com' \
    '*.twitter.com' \
  --priority 200 \
  --action Deny
```

Test:
```bash
kubectl exec egress-test -- curl -m 10 https://www.facebook.com
# Should be blocked
```

### 14. Enable Threat Intelligence
```bash
# Enable threat intelligence on firewall
az network firewall update \
  --resource-group $RESOURCE_GROUP \
  --name $FIREWALL_NAME \
  --threat-intel-mode Alert  # or Deny
```

### 15. View Defender for Containers Alerts
Check for alerts in Azure Security Center:

```bash
# List recent security alerts
az security alert list \
  --query "[?resourceIdentifier contains @, 'kubernetes'].{
    Name: alertDisplayName,
    Severity: severity,
    Status: status,
    Time: timeGeneratedUtc,
    Description: description
  }" \
  -o table
```

Common Defender alerts to expect:
- Privileged container detected
- Container with a sensitive volume mount detected
- Role binding to cluster-admin role detected
- Suspicious process detected
- Digital currency mining activity detected

### 16. Test Network Policy with Firewall
Combine network policies with firewall:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-lockdown
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: secure-app
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow to specific services only
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
```

### 17. Review Egress Traffic Patterns
```bash
# Query application rule logs
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "
    AzureFirewallApplicationRule
    | summarize Count = count() by Fqdn, Action
    | order by Count desc
    | take 20
  " \
  -o table

# Query network rule logs
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "
    AzureFirewallNetworkRule
    | summarize Count = count() by DestinationPort, Action
    | order by Count desc
  " \
  -o table
```

### 18. Create Alert Rule for Suspicious Activity
```bash
# Create alert for blocked egress attempts
az monitor metrics alert create \
  --name high-deny-rate \
  --resource-group $RESOURCE_GROUP \
  --scopes $(az network firewall show --resource-group $RESOURCE_GROUP --name $FIREWALL_NAME --query id -o tsv) \
  --condition "avg ApplicationRuleHit > 100" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --description "Alert when firewall denies many requests"
```

## Expected Results
- Azure Firewall routes all egress traffic
- Allowed destinations accessible (Microsoft services, registries)
- Blocked destinations timeout or fail
- Defender for Containers detects suspicious activities
- Firewall logs capture all egress attempts
- Threat intelligence blocks known malicious IPs
- Security alerts generated for policy violations

## Cleanup
```bash
kubectl delete pod egress-test suspicious-pod

# Remove firewall rules (optional)
az network firewall application-rule collection delete \
  --resource-group $RESOURCE_GROUP \
  --firewall-name $FIREWALL_NAME \
  --name block-social-media

# Disable Defender (optional)
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --disable-defender
```

## Key Takeaways
- **Azure Firewall** provides centralized egress control
- **User-Defined Routes (UDR)** force traffic through firewall
- **Application rules** filter by FQDN
- **Network rules** filter by IP/port
- **Defender for Containers** detects runtime threats
- Threat intelligence blocks known malicious sources
- Log Analytics provides egress visibility
- Combining network policies + firewall = defense in depth

## Architecture: Egress Control

```
┌──────────────┐
│  AKS Pod     │
└──────┬───────┘
       │ Egress traffic
       ▼
┌──────────────┐
│ Route Table  │ (0.0.0.0/0 → Firewall)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│Azure Firewall│
│ - App Rules  │
│ - Net Rules  │
│ - Threat Intel│
└──────┬───────┘
       │
       ▼ (Allowed)
    Internet
```

## Defender for Containers Capabilities

| Feature | Description |
|---------|-------------|
| Vulnerability scanning | Scans container images for CVEs |
| Runtime protection | Detects suspicious behavior |
| Kubernetes audit | Analyzes K8s audit logs |
| Network protection | Monitors egress patterns |
| Compliance | CIS benchmarks |

## Troubleshooting
- **Egress not working**: Check route table association
- **Firewall not blocking**: Verify rule priority and order
- **Defender not detecting**: Ensure profile is enabled
- **Logs not showing**: Check diagnostic settings

---



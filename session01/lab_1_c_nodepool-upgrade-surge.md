# Lab 1c: Node Pool Upgrade with Surge

## Objective
Perform a node pool upgrade using `--max-surge` and validate workload continuity.

## Prerequisites
- AKS cluster with at least one user node pool
- Application running on the node pool
- Azure CLI and `kubectl` configured

## Steps

### 1. Check Current Kubernetes Version
```bash
az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query "kubernetesVersion" -o tsv
```

List available upgrade versions:
```bash
az aks get-upgrades \
  --resource-group <resource-group> \
  --name <cluster-name> \
  -o table
```

Check node pool versions:
```bash
az aks nodepool list \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --query "[].{Name:name, Version:orchestratorVersion, VmSize:vmSize}" -o table
```

### 2. Deploy Test Application
Create a highly available deployment to test continuity during upgrade:

`ha-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-webapp
  namespace: default
spec:
  replicas: 6
  selector:
    matchLabels:
      app: ha-webapp
  template:
    metadata:
      labels:
        app: ha-webapp
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: ha-webapp
              topologyKey: kubernetes.io/hostname
      containers:
      - name: webapp
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
---
apiVersion: v1
kind: Service
metadata:
  name: ha-webapp-svc
spec:
  selector:
    app: ha-webapp
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ha-webapp-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: ha-webapp
```

Apply:
```bash
kubectl apply -f ha-deployment.yaml
kubectl get pods -l app=ha-webapp -o wide
kubectl get svc ha-webapp-svc
```

### 3. Monitor Application During Upgrade
In a separate terminal, continuously monitor the application:

```bash
# Monitor pods
kubectl get pods -l app=ha-webapp -w

# Or in another terminal, test availability
EXTERNAL_IP=$(kubectl get svc ha-webapp-svc -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
while true; do 
  curl -s -o /dev/null -w "%{http_code}\n" http://$EXTERNAL_IP
  sleep 1
done
```

### 4. Check Current Max Surge Setting
```bash
az aks nodepool show \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --query "upgradeSettings.maxSurge" -o tsv
```

### 5. Configure Max Surge
Update the node pool with max-surge setting for faster, safer upgrades:

```bash
az aks nodepool update \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --max-surge 33%
```

**Max Surge Options:**
- `1` or `33%`: One extra node at a time (slower, more conservative)
- `2` or `50%`: Two extra nodes (faster)
- `100%`: Double the nodes (fastest, highest cost during upgrade)

### 6. Perform Node Pool Upgrade
Upgrade the node pool to the next available version:

```bash
# Get target version
TARGET_VERSION="1.28.5"  # Replace with available version

# Start upgrade with max-surge
az aks nodepool upgrade \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --kubernetes-version $TARGET_VERSION \
  --max-surge 33%
```

### 7. Monitor Upgrade Progress
Track the upgrade process:

```bash
# Watch node status
kubectl get nodes -w

# Check node pool status
az aks nodepool show \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --query "provisioningState"

# View upgrade events
kubectl get events --sort-by='.lastTimestamp' | grep -i node
```

### 8. Observe Surge Behavior
During the upgrade, you should observe:

```bash
# Count nodes during upgrade
kubectl get nodes -l agentpool=<nodepool-name> --no-headers | wc -l

# Watch nodes being added and removed
kubectl get nodes -l agentpool=<nodepool-name> -o wide -w
```

**Expected behavior:**
- New nodes are added (surge nodes)
- Pods are drained from old nodes
- Old nodes are cordoned and deleted
- Process repeats until all nodes upgraded

### 9. Verify Application Availability
After upgrade completes:

```bash
# Check pod distribution
kubectl get pods -l app=ha-webapp -o wide

# Verify all pods are ready
kubectl get pods -l app=ha-webapp

# Check PDB status
kubectl get pdb ha-webapp-pdb

# Verify service accessibility
EXTERNAL_IP=$(kubectl get svc ha-webapp-svc -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXTERNAL_IP
```

### 10. Validate Upgrade Success
```bash
# Confirm new Kubernetes version
az aks nodepool show \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --query "orchestratorVersion" -o tsv

# Check all nodes are on new version
kubectl get nodes -l agentpool=<nodepool-name> -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion
```

### 11. Review Upgrade Logs
```bash
# Check Azure Activity Log
az monitor activity-log list \
  --resource-group <resource-group> \
  --max-events 50 \
  --query "[?contains(operationName.value, 'nodepool')].{Time:eventTimestamp, Operation:operationName.value, Status:status.value}" \
  -o table
```

## Expected Results
- Node pool successfully upgraded with zero or minimal downtime
- Application remained accessible throughout upgrade
- PodDisruptionBudget prevented excessive pod disruptions
- Max surge temporarily increased node count during upgrade
- All pods rescheduled successfully to new nodes

## Upgrade Strategy Comparison

| Max Surge | Speed | Cost During Upgrade | Risk | Use Case |
|-----------|-------|---------------------|------|----------|
| 1 (or 33%) | Slow | Low | Lowest | Production, cost-sensitive |
| 2 (or 50%) | Medium | Medium | Medium | Balanced approach |
| 100% | Fast | High | Higher | Dev/Test, time-critical |

## Cleanup
```bash
kubectl delete deployment ha-webapp
kubectl delete service ha-webapp-svc
kubectl delete pdb ha-webapp-pdb
```

## Key Takeaways
- **Max surge** controls how many extra nodes are created during upgrades
- Higher surge = faster upgrades but higher temporary costs
- **PodDisruptionBudgets (PDB)** ensure minimum pod availability during disruptions
- **Pod anti-affinity** spreads pods across nodes for better HA
- Node upgrades are rolling updates with cordon → drain → delete → replace
- Proper health checks (liveness/readiness) ensure smooth pod migrations

## Troubleshooting
- **Upgrade stuck**: Check PDB constraints, may be blocking drains
- **Pods not rescheduling**: Verify resource availability on new nodes
- **Downtime occurred**: Increase replica count or adjust PDB `minAvailable`
- **Upgrade failed**: Check Activity Logs for specific error messages

---



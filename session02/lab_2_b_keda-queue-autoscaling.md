# Lab 2b: KEDA Queue Autoscaling

## Objective
Deploy KEDA and observe event-driven scaling using Azure Queue Storage.

## Prerequisites
- AKS cluster running
- Azure Storage Account with Queue Storage
- Azure CLI and `kubectl` configured
- Helm 3 installed

## Steps

### 1. Create Azure Storage Queue
Create a storage account and queue:

```bash
# Set variables
RESOURCE_GROUP="<resource-group>"
STORAGE_ACCOUNT="kedademo$(openssl rand -hex 4)"
QUEUE_NAME="workitems"
LOCATION="eastus"

# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS

# Get connection string
CONNECTION_STRING=$(az storage account show-connection-string \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query connectionString -o tsv)

# Create queue
az storage queue create \
  --name $QUEUE_NAME \
  --connection-string "$CONNECTION_STRING"
```

### 2. Install KEDA
Add KEDA Helm repository and install:

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

kubectl create namespace keda

helm install keda kedacore/keda \
  --namespace keda \
  --version 2.12.0
```

Verify KEDA installation:
```bash
kubectl get pods -n keda
kubectl get crd | grep keda
```

Expected CRDs:
- `scaledobjects.keda.sh`
- `scaledjobs.keda.sh`
- `triggerauthentications.keda.sh`

### 3. Create Kubernetes Secret for Connection String
```bash
kubectl create secret generic azure-storage-secret \
  --from-literal=connection-string="$CONNECTION_STRING" \
  --namespace default
```

### 4. Deploy Queue Consumer Application
Create `queue-consumer-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue-consumer
  namespace: default
spec:
  replicas: 0  # KEDA will manage replicas
  selector:
    matchLabels:
      app: queue-consumer
  template:
    metadata:
      labels:
        app: queue-consumer
    spec:
      containers:
      - name: consumer
        image: mcr.microsoft.com/azure-cli
        command:
        - /bin/bash
        - -c
        - |
          while true; do
            echo "Checking queue at $(date)"
            MESSAGE=$(az storage message get \
              --queue-name ${QUEUE_NAME} \
              --connection-string "${CONNECTION_STRING}" \
              --query '[0].content' -o tsv)
            
            if [ ! -z "$MESSAGE" ]; then
              echo "Processing message: $MESSAGE"
              # Simulate processing
              sleep 10
              # Delete message
              az storage message delete \
                --id $(az storage message get \
                  --queue-name ${QUEUE_NAME} \
                  --connection-string "${CONNECTION_STRING}" \
                  --query '[0].id' -o tsv) \
                --pop-receipt $(az storage message get \
                  --queue-name ${QUEUE_NAME} \
                  --connection-string "${CONNECTION_STRING}" \
                  --query '[0].popReceipt' -o tsv) \
                --queue-name ${QUEUE_NAME} \
                --connection-string "${CONNECTION_STRING}"
              echo "Message processed and deleted"
            else
              echo "No messages in queue"
              sleep 5
            fi
          done
        env:
        - name: QUEUE_NAME
          value: "workitems"
        - name: CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: azure-storage-secret
              key: connection-string
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

Apply the deployment:
```bash
kubectl apply -f queue-consumer-deployment.yaml
```

### 5. Create KEDA ScaledObject
Create `keda-scaledobject.yaml`:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-consumer-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: queue-consumer
  minReplicaCount: 0
  maxReplicaCount: 10
  pollingInterval: 10  # Check queue every 10 seconds
  cooldownPeriod: 30   # Wait 30 seconds before scaling down
  triggers:
  - type: azure-queue
    metadata:
      queueName: workitems
      queueLength: "5"  # Scale up when >5 messages
      connectionFromEnv: CONNECTION_STRING
    authenticationRef:
      name: azure-queue-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-queue-auth
  namespace: default
spec:
  secretTargetRef:
  - parameter: connection
    name: azure-storage-secret
    key: connection-string
```

Apply the ScaledObject:
```bash
kubectl apply -f keda-scaledobject.yaml
```

### 6. Verify KEDA Configuration
```bash
kubectl get scaledobject
kubectl describe scaledobject queue-consumer-scaler

kubectl get triggerauthentication
kubectl describe triggerauthentication azure-queue-auth
```

Check initial pod count (should be 0):
```bash
kubectl get pods -l app=queue-consumer
kubectl get deployment queue-consumer
```

### 7. Add Messages to Queue
Populate the queue with messages:

```bash
# Add 20 messages
for i in {1..20}; do
  az storage message put \
    --queue-name $QUEUE_NAME \
    --content "Task-$i-$(date +%s)" \
    --connection-string "$CONNECTION_STRING"
  echo "Added message $i"
done

# Check queue length
az storage queue show \
  --name $QUEUE_NAME \
  --connection-string "$CONNECTION_STRING" \
  --query "approximateMessageCount"
```

### 8. Observe KEDA Autoscaling
Watch pods scale up automatically:

```bash
# Monitor deployment
kubectl get deployment queue-consumer -w

# Monitor pods
kubectl get pods -l app=queue-consumer -w

# Check HPA created by KEDA
kubectl get hpa
```

### 9. Monitor Queue Processing
View consumer logs:

```bash
kubectl logs -l app=queue-consumer -f --max-log-requests=10
```

Check queue message count decreasing:
```bash
watch -n 5 'az storage queue show \
  --name workitems \
  --connection-string "$CONNECTION_STRING" \
  --query "approximateMessageCount"'
```

### 10. Observe Scale Down
Once queue is empty, pods should scale down to 0:

```bash
kubectl get deployment queue-consumer -w
```

Check ScaledObject status:
```bash
kubectl get scaledobject queue-consumer-scaler -o yaml
```

### 11. Test Different Queue Lengths
Modify scaling threshold:

```bash
kubectl patch scaledobject queue-consumer-scaler \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/triggers/0/metadata/queueLength", "value": "2"}]'
```

Add more messages and observe faster scaling:
```bash
for i in {1..10}; do
  az storage message put \
    --queue-name $QUEUE_NAME \
    --content "HighPriority-Task-$i" \
    --connection-string "$CONNECTION_STRING"
done
```

### 12. View KEDA Metrics
```bash
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1 | jq .

kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/default/azure-queue-workitems" | jq .
```

### 13. Advanced: Use ScaledJob for Batch Processing
Create `keda-scaledjob.yaml`:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: queue-batch-processor
  namespace: default
spec:
  jobTargetRef:
    template:
      spec:
        containers:
        - name: processor
          image: mcr.microsoft.com/azure-cli
          command:
          - /bin/bash
          - -c
          - |
            MESSAGE=$(az storage message get \
              --queue-name ${QUEUE_NAME} \
              --connection-string "${CONNECTION_STRING}" \
              --query '[0].content' -o tsv)
            echo "Processing: $MESSAGE"
            sleep 15
        restartPolicy: Never
  pollingInterval: 10
  maxReplicaCount: 5
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 2
  triggers:
  - type: azure-queue
    metadata:
      queueName: workitems
      queueLength: "1"
      connectionFromEnv: CONNECTION_STRING
    authenticationRef:
      name: azure-queue-auth
```

## Expected Results
- KEDA installed and running in the cluster
- Deployment scales from 0 to multiple replicas based on queue length
- Messages are processed and removed from queue
- Pods scale down to 0 when queue is empty
- ScaledObject and TriggerAuthentication configured correctly
- Event-driven autoscaling responds within polling interval

## Cleanup
```bash
kubectl delete scaledobject queue-consumer-scaler
kubectl delete triggerauthentication azure-queue-auth
kubectl delete deployment queue-consumer
kubectl delete secret azure-storage-secret

# Delete queue and storage account
az storage queue delete --name $QUEUE_NAME --connection-string "$CONNECTION_STRING"
az storage account delete --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --yes

# Uninstall KEDA (optional)
helm uninstall keda -n keda
kubectl delete namespace keda
```

## Key Takeaways
- **KEDA** enables event-driven autoscaling based on external metrics
- Scales deployments to **zero** when no events are present
- **ScaledObject** defines scaling behavior and triggers
- **TriggerAuthentication** securely passes credentials
- Works with 50+ scalers: Azure Queue, Service Bus, Kafka, Redis, etc.
- More efficient than traditional HPA for event-driven workloads
- **ScaledJob** creates jobs instead of managing deployment replicas

## Comparison: KEDA vs HPA

| Feature | KEDA | HPA |
|---------|------|-----|
| Scale to zero | ✅ Yes | ❌ No (min 1) |
| External metrics | ✅ 50+ scalers | ⚠️ Limited |
| Event-driven | ✅ Native | ❌ Polling only |
| Jobs support | ✅ ScaledJob | ❌ No |

## Troubleshooting
- **Pods not scaling**: Check ScaledObject status and KEDA operator logs
- **Authentication errors**: Verify secret and TriggerAuthentication
- **Queue not detected**: Ensure connection string is correct
- **Slow scaling**: Adjust `pollingInterval` (default 30s)

---


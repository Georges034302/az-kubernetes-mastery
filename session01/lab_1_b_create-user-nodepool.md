# Lab 1b: Create User Node Pool

## Objective
Create a user node pool and schedule batch workloads using labels, taints, and tolerations.

## Prerequisites
- AKS cluster with at least one system node pool
- Azure CLI installed and configured
- `kubectl` configured

## Steps

### 1. Create User Node Pool
Create a dedicated node pool for batch workloads:

```bash
az aks nodepool add \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name batchpool \
  --node-count 2 \
  --node-vm-size Standard_D4s_v3 \
  --mode User \
  --labels workload=batch environment=production \
  --node-taints workload=batch:NoSchedule \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5
```

### 2. Verify Node Pool Creation
```bash
az aks nodepool list \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  -o table
```

Check nodes with labels:
```bash
kubectl get nodes --show-labels | grep batchpool
```

### 3. Inspect Node Taints
```bash
kubectl describe nodes -l agentpool=batchpool | grep -A 5 Taints
```

### 4. Deploy Workload Without Tolerations (Will Fail)
Create `batch-job-no-toleration.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-fail
spec:
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      nodeSelector:
        workload: batch
      containers:
      - name: processor
        image: busybox
        command: ["sh", "-c", "echo Processing batch data && sleep 30"]
      restartPolicy: Never
  backoffLimit: 2
```

Apply and observe:
```bash
kubectl apply -f batch-job-no-toleration.yaml
kubectl get pods -l app=batch-processor
kubectl describe pod -l app=batch-processor
```

**Expected:** Pod remains in `Pending` state due to taint.

### 5. Deploy Workload With Tolerations (Will Succeed)
Create `batch-job-with-toleration.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-success
spec:
  parallelism: 3
  completions: 3
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      nodeSelector:
        workload: batch
      tolerations:
      - key: workload
        operator: Equal
        value: batch
        effect: NoSchedule
      containers:
      - name: processor
        image: busybox
        command: 
        - sh
        - -c
        - |
          echo "Processing batch workload on node: $NODE_NAME"
          echo "Started at: $(date)"
          sleep 60
          echo "Completed at: $(date)"
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
      restartPolicy: Never
  backoffLimit: 3
```

Apply and monitor:
```bash
kubectl apply -f batch-job-with-toleration.yaml
kubectl get pods -l app=batch-processor -o wide
kubectl logs -l app=batch-processor -f
```

### 6. Verify Pod Placement
Confirm pods are running on the batch node pool:
```bash
kubectl get pods -l app=batch-processor -o wide
kubectl get nodes -l workload=batch
```

### 7. Deploy DaemonSet on Batch Nodes
Create `batch-monitoring-daemonset.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: batch-node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      nodeSelector:
        workload: batch
      tolerations:
      - key: workload
        operator: Equal
        value: batch
        effect: NoSchedule
      containers:
      - name: monitor
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            echo "Monitoring node: $NODE_NAME at $(date)"
            sleep 60
          done
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```

Apply:
```bash
kubectl apply -f batch-monitoring-daemonset.yaml
kubectl get daemonset batch-node-monitor
kubectl get pods -l app=node-monitor -o wide
```

### 8. Test Node Selector and Affinity
Create `advanced-scheduling.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-workload
spec:
  replicas: 2
  selector:
    matchLabels:
      app: batch-app
  template:
    metadata:
      labels:
        app: batch-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload
                operator: In
                values:
                - batch
      tolerations:
      - key: workload
        operator: Equal
        value: batch
        effect: NoSchedule
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

Apply and verify:
```bash
kubectl apply -f advanced-scheduling.yaml
kubectl get pods -l app=batch-app -o wide
```

## Expected Results
- User node pool created with custom labels and taints
- Pods without tolerations cannot schedule on tainted nodes
- Pods with correct tolerations successfully schedule on batch nodes
- DaemonSet deploys one pod per batch node
- Node affinity rules are respected

## Cleanup
```bash
kubectl delete job batch-job-fail batch-job-success
kubectl delete daemonset batch-node-monitor
kubectl delete deployment batch-workload

# Optional: Delete the node pool
az aks nodepool delete \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name batchpool
```

## Key Takeaways
- **Node pools** allow workload isolation and specialized compute
- **Taints** prevent pods from scheduling unless they have matching tolerations
- **Labels** enable node selection via `nodeSelector` or `nodeAffinity`
- **Tolerations** allow pods to "tolerate" node taints and schedule accordingly
- User node pools can autoscale independently from system pools

## Troubleshooting
- **Pods stuck in Pending**: Check taints/tolerations and node capacity
- **Node pool not visible**: Verify creation with `az aks nodepool list`
- **Autoscaler not working**: Ensure `--enable-cluster-autoscaler` was set

---



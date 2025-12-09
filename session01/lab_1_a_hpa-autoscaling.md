# Lab 1a: HPA Autoscaling

## Objective
Deploy a CPU-intensive workload and observe HPA-driven pod autoscaling.

## Prerequisites
- AKS cluster running
- `kubectl` configured
- Metrics Server enabled on the cluster

## Steps

### 1. Verify Metrics Server
```bash
kubectl get deployment metrics-server -n kube-system
```

If not installed, enable it:
```bash
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --enable-managed-identity \
  --enable-metrics-server
```

### 2. Deploy CPU-Intensive Application
Create a deployment file `cpu-stress-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-stress
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-stress
  template:
    metadata:
      labels:
        app: cpu-stress
    spec:
      containers:
      - name: cpu-stress
        image: containerstack/cpustress
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        command: ["/app/cpustress"]
        args: ["-cpus", "1"]
```

Apply the deployment:
```bash
kubectl apply -f cpu-stress-deployment.yaml
```

### 3. Create Horizontal Pod Autoscaler
```bash
kubectl autoscale deployment cpu-stress \
  --cpu-percent=50 \
  --min=1 \
  --max=10
```

Or use a YAML manifest `hpa.yaml`:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-stress-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-stress
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply:
```bash
kubectl apply -f hpa.yaml
```

### 4. Generate Load
Create a load generator pod:
```bash
kubectl run load-generator -it --rm --image=busybox -- /bin/sh

# Inside the pod
while true; do wget -q -O- http://cpu-stress.default.svc.cluster.local; done
```

### 5. Monitor HPA Behavior
Watch the HPA in action:
```bash
kubectl get hpa cpu-stress-hpa --watch
```

Check pod scaling:
```bash
kubectl get pods -l app=cpu-stress --watch
```

View detailed HPA status:
```bash
kubectl describe hpa cpu-stress-hpa
```

### 6. Observe Metrics
```bash
kubectl top pods -l app=cpu-stress
kubectl top nodes
```

## Expected Results
- HPA should scale pods from 1 to multiple replicas when CPU exceeds 50%
- Scaling should occur within 1-3 minutes of load increase
- Pods should scale down after load decreases (after cooldown period ~5 minutes)

## Cleanup
```bash
kubectl delete hpa cpu-stress-hpa
kubectl delete deployment cpu-stress
kubectl delete pod load-generator
```

## Key Takeaways
- HPA automatically scales pods based on observed CPU/memory metrics
- Proper resource requests are critical for HPA to function correctly
- Cooldown periods prevent rapid scaling fluctuations
- Metrics Server is required for HPA to collect pod metrics

## Troubleshooting
- **HPA shows `<unknown>` for metrics**: Metrics Server may not be running or pods lack resource requests
- **Pods not scaling**: Check if CPU threshold is actually being exceeded
- **Slow scaling**: Default HPA sync period is 15 seconds, scaling decisions take time

---

**Author:** Georges Bou Ghantous, Ph.D.

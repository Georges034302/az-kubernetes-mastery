# Lab 5b: Flagger Canary Progressive Delivery

## Objective
Implement progressive delivery with Flagger for automated canary deployments.

## Prerequisites
- AKS cluster running
- `kubectl` configured
- Helm 3 installed
- Ingress controller or service mesh (Istio/Linkerd) installed

## Steps

### 1. Install Flagger
```bash
# Add Flagger Helm repository
helm repo add flagger https://flagger.app
helm repo update

# Create namespace
kubectl create namespace flagger-system

# Install Flagger for Kubernetes (without service mesh)
helm install flagger flagger/flagger \
  --namespace flagger-system \
  --set meshProvider=kubernetes \
  --set metricsServer=http://prometheus.monitoring:9090

# For Istio integration:
# helm install flagger flagger/flagger \
#   --namespace flagger-system \
#   --set meshProvider=istio \
#   --set metricsServer=http://prometheus.istio-system:9090

# Verify installation
kubectl get pods -n flagger-system
```

### 2. Install Flagger Load Tester
```bash
# Install load tester for automated canary testing
helm install flagger-loadtester flagger/loadtester \
  --namespace flagger-system \
  --set cmd.timeout=1h

# Verify
kubectl get svc -n flagger-system
```

### 3. Deploy Application for Canary
Create `podinfo-deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: canary-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: canary-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfo
        image: ghcr.io/stefanprodan/podinfo:6.3.0
        ports:
        - containerPort: 9898
          name: http
          protocol: TCP
        - containerPort: 9797
          name: http-metrics
          protocol: TCP
        command:
        - ./podinfo
        - --port=9898
        - --level=info
        livenessProbe:
          httpGet:
            path: /healthz
            port: 9898
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 9898
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          limits:
            cpu: 1000m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: canary-demo
spec:
  type: ClusterIP
  selector:
    app: podinfo
  ports:
  - name: http
    port: 9898
    targetPort: http
    protocol: TCP
  - name: http-metrics
    port: 9797
    targetPort: http-metrics
    protocol: TCP
```

Apply:
```bash
kubectl apply -f podinfo-deployment.yaml

# Verify deployment
kubectl get pods -n canary-demo
kubectl get svc -n canary-demo
```

### 4. Create Canary Resource
Create `podinfo-canary.yaml`:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: canary-demo
spec:
  # Deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  
  # Service port
  service:
    port: 9898
    targetPort: 9898
    portDiscovery: true
  
  # Analysis configuration
  analysis:
    # Schedule interval
    interval: 30s
    # Max number of failed checks before rollback
    threshold: 5
    # Max traffic weight routed to canary
    maxWeight: 50
    # Canary increment step
    stepWeight: 10
    
    # Metrics for canary analysis
    metrics:
    - name: request-success-rate
      # Minimum req success rate (non 5xx responses)
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      # Maximum req duration P99
      thresholdRange:
        max: 500
      interval: 1m
    
    # Webhooks for load testing
    webhooks:
    - name: acceptance-test
      type: pre-rollout
      url: http://flagger-loadtester.flagger-system/
      timeout: 30s
      metadata:
        type: bash
        cmd: "curl -sd 'test' http://podinfo-canary.canary-demo:9898/token | grep token"
    - name: load-test
      type: rollout
      url: http://flagger-loadtester.flagger-system/
      timeout: 5s
      metadata:
        cmd: "hey -z 2m -q 10 -c 2 http://podinfo-canary.canary-demo:9898/"
```

Apply the canary:
```bash
kubectl apply -f podinfo-canary.yaml

# Watch canary status
kubectl get canary -n canary-demo -w

# Check events
kubectl describe canary podinfo -n canary-demo
```

### 5. Observe Canary Initialization
```bash
# Flagger will create primary deployment and service
kubectl get deployments -n canary-demo

# You should see:
# - podinfo (original)
# - podinfo-primary (stable version)
# - podinfo-canary will be created during rollout

# Check services
kubectl get svc -n canary-demo
```

### 6. Trigger Canary Deployment
Update the application to new version:

```bash
kubectl set image deployment/podinfo \
  podinfo=ghcr.io/stefanprodan/podinfo:6.3.1 \
  -n canary-demo
```

### 7. Monitor Canary Progression
```bash
# Watch canary status in real-time
kubectl get canary -n canary-demo -w

# View detailed events
kubectl describe canary podinfo -n canary-demo

# Check pod rollout
kubectl get pods -n canary-demo -w

# View Flagger logs
kubectl logs -n flagger-system deployment/flagger -f
```

Expected progression:
1. **Initialization**: Canary deployment created
2. **Progressing**: Traffic shifted incrementally (10%, 20%, 30%, 40%, 50%)
3. **Promoting**: All traffic to canary
4. **Finalizing**: Primary updated, canary scaled down
5. **Succeeded**: Rollout complete

### 8. Configure Advanced Metrics
Create custom metric template:

```yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: error-rate
  namespace: canary-demo
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring:9090
  query: |
    100 - sum(
      rate(
        http_requests_total{
          namespace="{{ namespace }}",
          deployment="{{ target }}",
          status!~"5.."
        }[{{ interval }}]
      )
    )
    /
    sum(
      rate(
        http_requests_total{
          namespace="{{ namespace }}",
          deployment="{{ target }}"
        }[{{ interval }}]
      )
    ) * 100
```

Update canary to use custom metric:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: canary-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    port: 9898
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: error-rate
      templateRef:
        name: error-rate
        namespace: canary-demo
      thresholdRange:
        max: 1
      interval: 1m
```

### 9. Implement A/B Testing
Create A/B testing canary:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo-ab
  namespace: canary-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    port: 9898
  analysis:
    interval: 1m
    threshold: 10
    iterations: 10
    match:
      # Route users with specific header to canary
      - headers:
          user-agent:
            regex: ".*Chrome.*"
      # Or based on cookie
      - headers:
          cookie:
            regex: "^(.*?;)?(canary=always)(;.*)?$"
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    webhooks:
    - name: load-test
      url: http://flagger-loadtester.flagger-system/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 -H 'Cookie: canary=always' http://podinfo-canary.canary-demo:9898/"
```

### 10. Configure Blue/Green Deployments
Create blue/green canary:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo-bluegreen
  namespace: canary-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    port: 9898
  analysis:
    interval: 1m
    threshold: 10
    # Blue/Green: 0% or 100% traffic
    iterations: 10
    # Skip gradual canary, only mirror and promote
    stepWeight: 100
    maxWeight: 100
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    webhooks:
    - name: smoke-test
      type: pre-rollout
      url: http://flagger-loadtester.flagger-system/
      timeout: 15s
      metadata:
        type: bash
        cmd: "curl -s http://podinfo-canary.canary-demo:9898/api/info | jq -r .version"
    - name: load-test
      url: http://flagger-loadtester.flagger-system/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://podinfo-canary.canary-demo:9898/"
```

### 11. Test Canary Rollback
Deploy a bad version to trigger rollback:

```bash
# Deploy version that fails health checks
kubectl set image deployment/podinfo \
  podinfo=ghcr.io/stefanprodan/podinfo:6.3.2 \
  -n canary-demo

# Set bad environment variable
kubectl set env deployment/podinfo PODINFO_UI_COLOR=bad \
  -n canary-demo

# Watch rollback
kubectl get canary podinfo -n canary-demo -w

# You should see status: Failed
# Primary deployment remains on stable version
```

### 12. Configure Alerting Webhooks
Add Slack notification:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: canary-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    port: 9898
  analysis:
    interval: 30s
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    alerts:
    - name: slack
      severity: info
      providerRef:
        name: slack
        namespace: flagger-system
```

Create alert provider:

```yaml
apiVersion: flagger.app/v1beta1
kind: AlertProvider
metadata:
  name: slack
  namespace: flagger-system
spec:
  type: slack
  channel: deployments
  username: flagger
  address: https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

### 13. Mirror Traffic for Testing
Configure traffic mirroring:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo-mirror
  namespace: canary-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    port: 9898
  analysis:
    interval: 1m
    threshold: 5
    # Mirror 100% of traffic to canary
    mirror: true
    mirrorWeight: 100
    iterations: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    webhooks:
    - name: load-test
      url: http://flagger-loadtester.flagger-system/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://podinfo.canary-demo:9898/"
```

### 14. Session Affinity
Configure session affinity for stateful apps:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo-session
  namespace: canary-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    port: 9898
    sessionAffinity:
      cookieName: flagger-cookie
      maxAge: 86400
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
```

### 15. Multi-Port Services
Handle multiple ports:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo-multiport
  namespace: canary-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    # Primary port
    port: 9898
    # Additional ports
    ports:
    - name: metrics
      port: 9797
      targetPort: 9797
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
```

### 16. Canary with HPA
Combine with Horizontal Pod Autoscaler:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: canary-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
---
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: canary-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  # Flagger will manage HPA for both primary and canary
  autoscalerRef:
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    name: podinfo
  service:
    port: 9898
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
```

### 17. Query Canary Metrics
```bash
# Get canary status
kubectl get canary -n canary-demo

# Describe canary for events
kubectl describe canary podinfo -n canary-demo

# Check primary deployment
kubectl get deployment podinfo-primary -n canary-demo

# View canary deployment (during rollout)
kubectl get deployment podinfo -n canary-demo

# Check services
kubectl get svc -n canary-demo
```

### 18. Pause/Resume Canary
```bash
# Pause canary analysis
kubectl annotate canary podinfo \
  flagger.app/skip-analysis=true \
  -n canary-demo

# Resume canary analysis
kubectl annotate canary podinfo \
  flagger.app/skip-analysis- \
  -n canary-demo
```

### 19. Monitor with Grafana
Install Flagger Grafana dashboard:

```bash
# Add Grafana dashboards
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/kustomize/grafana/dashboards.yaml

# Access Grafana and import dashboard ID: 15513
```

### 20. Cleanup
```bash
# Delete canary
kubectl delete canary podinfo -n canary-demo

# Delete application
kubectl delete -f podinfo-deployment.yaml

# Delete namespace
kubectl delete namespace canary-demo

# Uninstall Flagger
helm uninstall flagger -n flagger-system
helm uninstall flagger-loadtester -n flagger-system
kubectl delete namespace flagger-system
```

## Expected Results
- Flagger installed and managing canary deployments
- Automated progressive traffic shifting
- Metrics-based canary analysis
- Automatic rollback on failure detection
- Load testing integrated into rollout process
- Primary deployment always stable
- Zero-downtime deployments

## Key Takeaways
- **Flagger** automates progressive delivery strategies
- **Canary deployments** gradually shift traffic to new version
- **Metrics analysis** validates canary before promotion
- **Automatic rollback** on threshold violations
- **Load testing** validates performance during rollout
- **A/B testing** routes based on headers/cookies
- **Blue/Green** provides instant switching
- **Traffic mirroring** tests without impacting users
- Works with Kubernetes, Istio, Linkerd, App Mesh
- Integrates with Prometheus for metrics

## Flagger Deployment Strategies

| Strategy | Traffic Pattern | Use Case |
|----------|-----------------|----------|
| Canary | Gradual 0→100% | Standard progressive delivery |
| A/B Testing | Header/cookie routing | Feature testing |
| Blue/Green | Instant 0→100% | Quick rollover |
| Mirroring | Shadow traffic | Testing without risk |

## Troubleshooting
- **Canary stuck in Progressing**: Check metrics thresholds
- **Immediate rollback**: Verify health checks and metrics
- **No traffic shift**: Ensure service mesh/ingress configured
- **Webhook failures**: Check loadtester connectivity
- **Metrics unavailable**: Verify Prometheus integration

---

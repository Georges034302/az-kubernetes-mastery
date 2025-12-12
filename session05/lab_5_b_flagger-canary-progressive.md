# Lab 5b: Flagger Canary Progressive Delivery

## Objective
Learn to implement automated, metrics-driven progressive delivery on AKS using Flagger for canary deployments with automated analysis and rollback.

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Helm 3 installed
- Docker installed (optional, for custom apps)
- Basic understanding of Kubernetes deployments

---

## Lab Parameters

```bash
# Resource and location settings
RESOURCE_GROUP="rg-aks-flagger-lab"
LOCATION="australiaeast"  # Sydney, Australia

# AKS cluster settings
CLUSTER_NAME="aks-flagger-lab"
NODE_COUNT=3
NODE_SIZE="Standard_D2s_v3"
K8S_VERSION="1.29"

# Flagger settings
FLAGGER_NS="flagger-system"
CANARY_NS="canary-demo"
PROMETHEUS_NS="monitoring"

# Application settings
APP_NAME="podinfo"
PODINFO_VERSION="6.3.0"
PODINFO_NEW_VERSION="6.3.1"
PROMETHEUS_URL="http://prometheus.${PROMETHEUS_NS}:9090"
```

Display configuration:
```bash
echo "=== Lab 5b: Flagger Canary Progressive Delivery ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Node Count: $NODE_COUNT"
echo "Flagger Namespace: $FLAGGER_NS"
echo "Canary Namespace: $CANARY_NS"
echo "Prometheus Namespace: $PROMETHEUS_NS"
echo "Application: $APP_NAME"
echo "Prometheus URL: $PROMETHEUS_URL"
```

---

## Step 1: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \              # `Resource group name`
  --location $LOCATION                  # `Azure region`
```

---

## Step 2: Create AKS Cluster

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \            # `Resource group`
  --name $CLUSTER_NAME \                        # `Cluster name`
  --location $LOCATION \                        # `Azure region`
  --node-count $NODE_COUNT \                    # `Number of nodes`
  --node-vm-size $NODE_SIZE \                   # `VM size for nodes`
  --kubernetes-version $K8S_VERSION \           # `Kubernetes version`
  --network-plugin azure \                      # `Azure CNI networking`
  --enable-managed-identity \                   # `Use managed identity`
  --generate-ssh-keys                           # `Generate SSH keys`
```

Get credentials:
```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing  # `Merge credentials to kubeconfig`
```

---

## Step 3: Install Prometheus

Create namespace:
```bash
kubectl create namespace $PROMETHEUS_NS  # `Create monitoring namespace`
```

Add Prometheus Helm repo:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts  # `Add Prometheus repo`
helm repo update  # `Update repo index`
```

Install Prometheus:
```bash
helm install prometheus prometheus-community/prometheus \
  --namespace $PROMETHEUS_NS \          # `Target namespace`
  --set alertmanager.enabled=false \    # `Disable Alertmanager`
  --set pushgateway.enabled=false \     # `Disable Pushgateway`
  --set nodeExporter.enabled=false \    # `Disable Node Exporter`
  --wait                                # `Wait for deployment`
```

Verify Prometheus:
```bash
kubectl get pods --namespace $PROMETHEUS_NS  # `List Prometheus pods`
kubectl get svc --namespace $PROMETHEUS_NS   # `Show Prometheus services`
```

---

## Step 4: Install Flagger

Add Flagger Helm repository:
```bash
helm repo add flagger https://flagger.app  # `Add Flagger Helm repo`
helm repo update  # `Update repo index`
```

Create Flagger namespace:
```bash
kubectl create namespace $FLAGGER_NS  # `Create Flagger system namespace`
```

Install Flagger:
```bash
helm install flagger flagger/flagger \
  --namespace $FLAGGER_NS \             # `Target namespace`
  --set meshProvider=kubernetes \       # `Use Kubernetes provider (no service mesh)`
  --set metricsServer=$PROMETHEUS_URL \ # `Prometheus metrics endpoint`
  --wait                                # `Wait for deployment`
```

Verify Flagger installation:
```bash
kubectl get pods --namespace $FLAGGER_NS  # `List Flagger pods`
kubectl get crd | grep flagger            # `Show Flagger CRDs`
```

---

## Step 5: Install Flagger Load Tester

Install load tester:
```bash
helm install flagger-loadtester flagger/loadtester \
  --namespace $FLAGGER_NS \     # `Target namespace`
  --set cmd.timeout=1h \        # `Test timeout duration`
  --wait                        # `Wait for deployment`
```

Verify load tester:
```bash
kubectl get svc --namespace $FLAGGER_NS  # `Show load tester service`
```

---

## Step 6: Deploy Application for Canary

Create canary namespace:
```bash
kubectl create namespace $CANARY_NS  # `Create canary demo namespace`
```

Deploy podinfo application:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME
  namespace: $CANARY_NS
spec:
  replicas: 2
  selector:
    matchLabels:
      app: $APP_NAME
  template:
    metadata:
      labels:
        app: $APP_NAME
    spec:
      containers:
      - name: $APP_NAME
        image: ghcr.io/stefanprodan/podinfo:$PODINFO_VERSION
        ports:
        - containerPort: 9898
          name: http
        - containerPort: 9797
          name: http-metrics
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
  name: $APP_NAME
  namespace: $CANARY_NS
spec:
  type: ClusterIP
  selector:
    app: $APP_NAME
  ports:
  - name: http
    port: 9898
    targetPort: http
  - name: http-metrics
    port: 9797
    targetPort: http-metrics
EOF
```

Verify deployment:
```bash
kubectl get pods --namespace $CANARY_NS  # `List application pods`
kubectl get svc --namespace $CANARY_NS   # `Show services`

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

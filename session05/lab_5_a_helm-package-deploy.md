# Lab 5a: Helm Package and Deploy

## Objective
Create, package, and deploy a custom Helm chart to AKS with a containerized application using Azure Container Registry (ACR).

## Prerequisites
- Azure CLI installed and authenticated
- `kubectl` installed
- Docker installed
- Basic understanding of Kubernetes manifests

---

## Lab Parameters

```bash
# Resource and location settings
RESOURCE_GROUP="rg-aks-helm-lab"
LOCATION="australiaeast"  # Sydney, Australia

# AKS cluster settings
CLUSTER_NAME="aks-helm-lab"
NODE_COUNT=2
NODE_SIZE="Standard_D2s_v3"
K8S_VERSION="1.29"

# Azure Container Registry settings
ACR_NAME="acrhelm$RANDOM"

# Application settings
APP_NAME="myapp"
APP_NAMESPACE="helm-demo"
IMAGE_TAG="1.0.0"

# Helm chart settings
CHART_NAME="myapp"
CHART_VERSION="1.0.0"
RELEASE_NAME="myapp-release"
```

Display configuration:
```bash
echo "=== Lab 5a: Helm Package and Deploy ==="
echo "Resource Group: $RESOURCE_GROUP"
echo "Location: $LOCATION"
echo "Cluster Name: $CLUSTER_NAME"
echo "Node Count: $NODE_COUNT"
echo "Node Size: $NODE_SIZE"
echo "Kubernetes Version: $K8S_VERSION"
echo "ACR Name: $ACR_NAME"
echo "App Name: $APP_NAME"
echo "Helm Chart: $CHART_NAME"
echo "Release Name: $RELEASE_NAME"
```

---

## Step 1: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \              # `Resource group name`
  --location $LOCATION                  # `Azure region`
```

---

## Step 2: Create Azure Container Registry

```bash
az acr create \
  --resource-group $RESOURCE_GROUP \    # `Resource group`
  --name $ACR_NAME \                    # `ACR name (globally unique)`
  --sku Standard \                      # `SKU tier`
  --admin-enabled true                  # `Enable admin user`
```

Capture ACR login server:
```bash
ACR_LOGIN_SERVER=$(az acr show \
  --name $ACR_NAME \
  --query loginServer \
  --output tsv)  # `Get ACR FQDN`

echo "ACR Login Server: $ACR_LOGIN_SERVER"
```

---

## Step 3: Create AKS Cluster

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
  --attach-acr $ACR_NAME \                      # `Attach ACR for image pull`
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

## Step 4: Verify Helm Installation

```bash
if ! command -v helm &> /dev/null; then
  curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
fi

helm version --short  # `Verify Helm installation`
```

---

## Step 5: Build Application Container Image

Create application directory:
```bash
mkdir -p $APP_NAME
cd $APP_NAME
```

Create `app.py`:
```bash
cat <<'EOF' > app.py
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify({
        "message": "Hello from MyApp via Helm!",
        "hostname": socket.gethostname(),
        "version": os.getenv("APP_VERSION", "1.0.0")
    })

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
EOF
```

Create `Dockerfile`:
```bash
cat <<'EOF' > Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY app.py .

RUN pip install --no-cache-dir flask

EXPOSE 5000

ENV APP_VERSION=1.0.0

CMD ["python", "app.py"]
EOF
```

---

## Step 6: Build and Push Image to ACR

Login to ACR:
```bash
az acr login --name $ACR_NAME  # `Authenticate with ACR`
```

Build Docker image:
```bash
docker build \
  --tag $ACR_LOGIN_SERVER/$APP_NAME:$IMAGE_TAG \  # `Tag with ACR FQDN`
  --file Dockerfile \                              # `Dockerfile path`
  .                                                # `Build context`
```

Push image to ACR:
```bash
docker push $ACR_LOGIN_SERVER/$APP_NAME:$IMAGE_TAG  # `Push to ACR`
```

---

## Step 7: Create Helm Chart

Generate chart scaffold:
```bash
cd ..
helm create $CHART_NAME  # `Create new Helm chart`
```

---

## Step 8: Customize Chart Metadata

Update `$CHART_NAME/Chart.yaml`:
```bash
cat <<EOF > $CHART_NAME/Chart.yaml
apiVersion: v2
name: $CHART_NAME
description: Flask web application Helm chart for AKS
type: application
version: $CHART_VERSION
appVersion: "$IMAGE_TAG"
keywords:
  - web
  - flask
  - python
maintainers:
  - name: AKS Admin
home: https://github.com/your-org/$CHART_NAME
EOF
```

---

## Step 9: Configure Chart Values

Update `$CHART_NAME/values.yaml`:
```bash
cat <<EOF > $CHART_NAME/values.yaml
replicaCount: 3

image:
  repository: $ACR_LOGIN_SERVER/$APP_NAME
  pullPolicy: IfNotPresent
  tag: "$IMAGE_TAG"

serviceAccount:
  create: true
  name: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 5000

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: false

livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 5
EOF
```

---

## Step 10: Validate Helm Chart

Lint the chart:
```bash
helm lint $CHART_NAME  # `Check chart for issues`
```

Render templates locally:
```bash
helm template $RELEASE_NAME $CHART_NAME \
  --namespace $APP_NAMESPACE \  # `Target namespace`
  > rendered-manifests.yaml     # `Save output to file`
```

Dry-run installation:
```bash
helm install $RELEASE_NAME $CHART_NAME \
  --namespace $APP_NAMESPACE \  # `Target namespace`
  --create-namespace \           # `Create namespace if missing`
  --dry-run \                    # `Simulate installation`
  --debug                        # `Show debug output`
```

---

## Step 11: Install Helm Release

Install the chart:
```bash
helm install $RELEASE_NAME $CHART_NAME \
  --namespace $APP_NAMESPACE \          # `Target namespace`
  --create-namespace \                  # `Create namespace if missing`
  --wait \                              # `Wait for resources to be ready`
  --timeout 5m                          # `Timeout duration`
```

Check release status:
```bash
helm status $RELEASE_NAME \
  --namespace $APP_NAMESPACE  # `Show release details`
```

List releases:
```bash
helm list --namespace $APP_NAMESPACE  # `List all releases in namespace`
```

---

## Step 12: Verify Application Deployment

Check pods:
```bash
kubectl get pods --namespace $APP_NAMESPACE  # `List pods in namespace`
```

Check service:
```bash
kubectl get service --namespace $APP_NAMESPACE  # `Show service details`
```

Test application:
```bash
kubectl port-forward \
  --namespace $APP_NAMESPACE \              # `Target namespace`
  svc/$RELEASE_NAME-$CHART_NAME 8080:80     # `Forward local 8080 to service port 80`
```

Open browser to `http://localhost:8080` or:
```bash
curl http://localhost:8080  # `Test application endpoint`
```

---

## Step 13: Upgrade Helm Release

Upgrade with new replica count:
```bash
helm upgrade $RELEASE_NAME $CHART_NAME \
  --namespace $APP_NAMESPACE \          # `Target namespace`
  --set replicaCount=5 \                # `Override replica count`
  --wait                                # `Wait for upgrade to complete`
```

View release history:
```bash
helm history $RELEASE_NAME --namespace $APP_NAMESPACE  # `Show all revisions`
```

---

## Step 14: Rollback Helm Release

Rollback to previous revision:
```bash
helm rollback $RELEASE_NAME \
  --namespace $APP_NAMESPACE \          # `Target namespace`
  --wait                                # `Wait for rollback to complete`
```

Rollback to specific revision:
```bash
helm rollback $RELEASE_NAME 1 \       # `Revision number to rollback to`
  --namespace $APP_NAMESPACE           # `Target namespace`
```

---

## Step 15: Package Helm Chart

Package the chart:
```bash
helm package $CHART_NAME  # `Create .tgz archive`

ls -lh $CHART_NAME-$CHART_VERSION.tgz  # `Show packaged file`
```

---

## Step 16: Create Helm Repository (Optional)

Create local Helm repository:
```bash
mkdir -p helm-repo  # `Create repository directory`
mv $CHART_NAME-$CHART_VERSION.tgz helm-repo/  # `Move chart to repo`

helm repo index helm-repo/ \
  --url https://example.com/charts  # `Generate index.yaml`
```

---

## Expected Results

After completing this lab, you should have:

- ✅ AKS cluster running in Australia East
- ✅ Azure Container Registry with custom application image
- ✅ Flask application containerized and pushed to ACR
- ✅ Custom Helm chart created with proper structure
- ✅ Chart validated with `helm lint`
- ✅ Application deployed via Helm release
- ✅ Release upgraded with new configuration
- ✅ Release rolled back to previous version
- ✅ Helm chart packaged as `.tgz` archive
- ✅ Local Helm repository created

---

## Cleanup

### Option 1: Delete Helm Release Only

Uninstall Helm release:
```bash
helm uninstall $RELEASE_NAME --namespace $APP_NAMESPACE  # `Remove release`
```

Delete namespace:
```bash
kubectl delete namespace $APP_NAMESPACE  # `Delete namespace and resources`
```

### Option 2: Delete All Azure Resources

Delete resource group:
```bash
az group delete \
  --name $RESOURCE_GROUP \  # `Resource group name`
  --yes \                   # `Skip confirmation`
  --no-wait                 # `Don't wait for completion`
```

Clean up local files:
```bash
cd ..
rm -rf $APP_NAME $CHART_NAME helm-repo rendered-manifests.yaml  # `Remove local artifacts`
```

---

## How This Connects to Kubernetes Package Management

### Helm Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Helm Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐                                               │
│  │  Helm CLI    │                                               │
│  │  (Local)     │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         │ helm install / upgrade / rollback                     │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────┐       │
│  │           Helm Chart Package (.tgz)                 │       │
│  │  ┌──────────────────────────────────────────────┐  │       │
│  │  │ Chart.yaml (metadata)                        │  │       │
│  │  │ values.yaml (configuration)                  │  │       │
│  │  │ templates/ (K8s manifests with {{ }} syntax)│  │       │
│  │  │ charts/ (dependencies)                       │  │       │
│  │  └──────────────────────────────────────────────┘  │       │
│  └───────────────────┬─────────────────────────────────┘       │
│                      │                                          │
│                      │ Render templates with values             │
│                      ▼                                          │
│  ┌─────────────────────────────────────────────────────┐       │
│  │          Generated Kubernetes Manifests             │       │
│  │  (Deployment, Service, ConfigMap, Secret, etc.)     │       │
│  └───────────────────┬─────────────────────────────────┘       │
│                      │                                          │
│                      │ kubectl apply                            │
│                      ▼                                          │
│  ┌─────────────────────────────────────────────────────┐       │
│  │           Kubernetes API Server (AKS)               │       │
│  │  ┌────────────────────────────────────────────┐    │       │
│  │  │  Release: myapp-v1 (Revision 1)            │    │       │
│  │  │  • Deployment: myapp (3 replicas)          │    │       │
│  │  │  • Service: myapp-svc                      │    │       │
│  │  │  • ConfigMap: myapp-config                 │    │       │
│  │  └────────────────────────────────────────────┘    │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Helm Chart Structure Explained

```
myapp-chart/
├── Chart.yaml          # Chart metadata (name, version, dependencies)
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts (subcharts)
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml # Deployment with {{ .Values.* }} placeholders
│   ├── service.yaml    # Service template
│   ├── _helpers.tpl    # Named template definitions (reusable)
│   └── NOTES.txt       # Post-installation instructions
└── .helmignore         # Files to exclude from package
```

### Helm Template Syntax Examples

**Variable Substitution:**
```yaml
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
replicas: {{ .Values.replicaCount }}
```

**Conditionals:**
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
{{- end }}
```

**Loops:**
```yaml
{{- range $key, $value := .Values.configMap.data }}
{{ $key }}: {{ $value | quote }}
{{- end }}
```

**Named Templates:**
```yaml
{{- include "myapp.fullname" . }}  # Calls _helpers.tpl template
```

### Helm Release Lifecycle

```
┌──────────┐     helm install      ┌──────────┐
│  Chart   │ ─────────────────────▶│ Release  │
│ Package  │                        │ (Rev 1)  │
└──────────┘                        └─────┬────┘
                                          │
                helm upgrade              │
                (new values)              │
                         │                │
                         ▼                ▼
                    ┌──────────┐     ┌──────────┐
                    │ Release  │     │ Release  │
                    │ (Rev 2)  │────▶│ (Rev 3)  │
                    └─────┬────┘     └─────┬────┘
                          │                │
                          │ helm rollback  │
                          └────────────────┘
```

### Helm vs kubectl Comparison

| Feature | kubectl | Helm |
|---------|---------|------|
| **Manifest Management** | Individual YAML files | Packaged charts |
| **Configuration** | Hardcoded values | values.yaml + templating |
| **Versioning** | Manual Git tags | Built-in chart versioning |
| **Rollback** | Manual reapply | `helm rollback` (automated) |
| **Dependencies** | Manual tracking | Defined in Chart.yaml |
| **Release History** | No built-in tracking | `helm history` command |
| **Package Distribution** | Git/URL download | Helm repositories |
| **Templating** | Static YAML | Go template engine |

### Chart Repository Concepts

```
┌────────────────────────────────────────────────────────────┐
│                 Helm Repository Structure                   │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  https://charts.example.com/                               │
│  ├── index.yaml         (repository index)                 │
│  ├── myapp-1.0.0.tgz    (chart package v1.0.0)            │
│  ├── myapp-1.1.0.tgz    (chart package v1.1.0)            │
│  └── otherapp-2.0.0.tgz (another chart)                    │
│                                                             │
│  index.yaml contains:                                       │
│  apiVersion: v1                                            │
│  entries:                                                   │
│    myapp:                                                   │
│      - name: myapp                                         │
│        version: 1.1.0                                      │
│        urls:                                               │
│          - https://charts.example.com/myapp-1.1.0.tgz     │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

**Repository Workflow:**
1. `helm repo add myrepo https://charts.example.com/`
2. `helm repo update` (fetch latest index.yaml)
3. `helm search repo myapp`
4. `helm install release-name myrepo/myapp`

### Key Helm Concepts

| Concept | Description |
|---------|-------------|
| **Chart** | Package containing K8s templates + metadata + default values |
| **Release** | Installed instance of a chart (has unique name + revision history) |
| **Repository** | Collection of charts with index.yaml for discovery |
| **Values** | Configuration parameters (values.yaml + --set overrides) |
| **Template** | K8s manifest with Go template placeholders ({{ .Values.* }}) |
| **Revision** | Each install/upgrade creates new revision (enables rollback) |
| **Dependency** | Charts can depend on other charts (defined in Chart.yaml) |
| **Hook** | Special resources executed at specific points (pre-install, post-upgrade) |

---

## Key Takeaways

- ✅ **Helm simplifies Kubernetes deployment** by packaging manifests, configuration, and dependencies into reusable charts
- ✅ **Templating enables configuration flexibility** without modifying YAML files directly (values.yaml + --set flags)
- ✅ **Release management provides versioning and rollback** capabilities crucial for production environments
- ✅ **Chart repositories centralize distribution** similar to package managers (apt, yum, npm)
- ✅ **Azure Container Registry stores custom images** referenced by Helm chart deployments
- ✅ **Health probes in values.yaml** ensure Kubernetes monitors application readiness (/health endpoint)
- ✅ **Helm lint validates charts** before deployment, catching template errors early
- ✅ **helm upgrade --set overrides** enable dynamic configuration changes without editing values.yaml
- ✅ **Revision history tracking** allows auditing and rollback to any previous release state
- ✅ **Chart packaging (.tgz files)** standardizes distribution and versioning across teams

---

## Commands Quick Reference

| Command | Purpose |
|---------|---------|
| `helm create <name>` | Generate new chart scaffold |
| `helm lint <chart>` | Validate chart for issues |
| `helm template <name> <chart>` | Render templates locally (dry-run) |
| `helm install <name> <chart>` | Deploy chart as new release |
| `helm upgrade <name> <chart>` | Update existing release |
| `helm rollback <name> <revision>` | Revert to previous revision |
| `helm uninstall <name>` | Delete release and resources |
| `helm list` | Show all releases |
| `helm history <name>` | View release revision history |
| `helm get values <name>` | Show values used in release |
| `helm package <chart>` | Create .tgz archive |
| `helm repo index <dir>` | Generate repository index.yaml |
| `helm repo add <name> <url>` | Add chart repository |
| `helm search repo <keyword>` | Search repositories |

---

## Troubleshooting

### Issue: Helm installation fails with "release already exists"

**Solution:**
```bash
helm list --all-namespaces  # `Find existing release`
helm uninstall $RELEASE_NAME --namespace $APP_NAMESPACE
```

### Issue: Pods not pulling image from ACR

**Symptom:** `ErrImagePull` or `ImagePullBackOff`

**Solution:**
```bash
# Verify ACR integration
az aks check-acr \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --acr $ACR_NAME

# Reattach ACR if needed
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --attach-acr $ACR_NAME
```

### Issue: Template rendering errors

**Solution:**
```bash
# Debug template rendering
helm template $RELEASE_NAME $CHART_NAME --debug

# Check values
helm get values $RELEASE_NAME --namespace $APP_NAMESPACE
```

### Issue: Rollback fails

**Solution:**
```bash
# Check revision history
helm history $RELEASE_NAME --namespace $APP_NAMESPACE

# Force rollback
helm rollback $RELEASE_NAME <revision> \
  --namespace $APP_NAMESPACE \
  --force
```

### Issue: Chart validation errors

**Solution:**
```bash
# Lint with verbose output
helm lint $CHART_NAME --debug

# Check Chart.yaml syntax
cat $CHART_NAME/Chart.yaml

# Validate values.yaml
helm template $CHART_NAME --validate
```

---

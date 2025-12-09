# Lab 5a: Helm Package and Deploy

## Objective
Package and deploy applications using Helm charts.

## Prerequisites
- AKS cluster running
- `kubectl` configured
- Helm 3 installed
- Basic understanding of Kubernetes manifests

## Steps

### 1. Install Helm
```bash
# Check if Helm is already installed
helm version

# If not installed, install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version --short
```

### 2. Add Helm Repositories
```bash
# Add official stable charts repository
helm repo add stable https://charts.helm.sh/stable

# Add Bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Add Azure Marketplace repository
helm repo add azure-marketplace https://marketplace.azurecr.io/helm/v1/repo

# Update repositories
helm repo update

# List all repositories
helm repo list
```

### 3. Search for Charts
```bash
# Search for nginx charts
helm search repo nginx

# Search for specific version
helm search repo nginx --version 13.2.0

# Search in all repositories
helm search hub wordpress
```

### 4. Install a Chart from Repository
```bash
# Install Redis
helm install my-redis bitnami/redis \
  --namespace default \
  --create-namespace

# Check installation status
helm status my-redis

# List installed releases
helm list

# Get values used in installation
helm get values my-redis
```

### 5. Create Your Own Helm Chart
```bash
# Create a new chart
helm create myapp

# Explore the chart structure
tree myapp/
```

Chart structure:
```
myapp/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── charts/             # Dependencies
├── templates/          # Kubernetes manifests templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── tests/
└── .helmignore
```

### 6. Customize Chart.yaml
Edit `myapp/Chart.yaml`:

```yaml
apiVersion: v2
name: myapp
description: A production-grade web application Helm chart
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - web
  - nginx
  - kubernetes
maintainers:
  - name: Georges Bou Ghantous
    email: your-email@example.com
home: https://github.com/your-org/myapp
sources:
  - https://github.com/your-org/myapp
dependencies: []
```

### 7. Configure values.yaml
Edit `myapp/values.yaml`:

```yaml
replicaCount: 3

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25-alpine"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9113"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false

service:
  type: ClusterIP
  port: 80
  targetPort: 80
  annotations: {}

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - myapp
          topologyKey: kubernetes.io/hostname

livenessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5

configMap:
  enabled: true
  data:
    LOG_LEVEL: "info"
    ENVIRONMENT: "production"

secret:
  enabled: true
  data:
    DATABASE_PASSWORD: "changeme"
    API_KEY: "secret-key"
```

### 8. Create Custom Templates
Create `myapp/templates/configmap.yaml`:

```yaml
{{- if .Values.configMap.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.configMap.data }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

Create `myapp/templates/secret.yaml`:

```yaml
{{- if .Values.secret.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
type: Opaque
data:
  {{- range $key, $value := .Values.secret.data }}
  {{ $key }}: {{ $value | b64enc | quote }}
  {{- end }}
{{- end }}
```

### 9. Update Deployment Template
Edit `myapp/templates/deployment.yaml` to reference ConfigMap and Secret:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 12 }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        envFrom:
        {{- if .Values.configMap.enabled }}
        - configMapRef:
            name: {{ include "myapp.fullname" . }}
        {{- end }}
        {{- if .Values.secret.enabled }}
        - secretRef:
            name: {{ include "myapp.fullname" . }}
        {{- end }}
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 12 }}
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 12 }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 10. Validate the Chart
```bash
# Lint the chart
helm lint myapp/

# Dry-run installation
helm install myapp-test myapp/ --dry-run --debug

# Template the chart to see generated manifests
helm template myapp myapp/ > rendered.yaml

# View specific template
helm template myapp myapp/ --show-only templates/deployment.yaml
```

### 11. Package the Chart
```bash
# Package the chart
helm package myapp/

# This creates: myapp-1.0.0.tgz

# Verify package contents
tar -tzf myapp-1.0.0.tgz
```

### 12. Install the Chart
```bash
# Install with default values
helm install myapp ./myapp

# Install with custom values
helm install myapp ./myapp \
  --set replicaCount=5 \
  --set image.tag=1.26-alpine

# Install with values file
cat > custom-values.yaml <<EOF
replicaCount: 2
image:
  tag: "1.24-alpine"
ingress:
  enabled: false
resources:
  limits:
    cpu: 500m
    memory: 512Mi
EOF

helm install myapp-custom ./myapp -f custom-values.yaml
```

### 13. Manage Chart Releases
```bash
# List all releases
helm list

# Get release status
helm status myapp

# Get release history
helm history myapp

# Get all values for a release
helm get values myapp

# Get generated manifests
helm get manifest myapp
```

### 14. Upgrade a Release
```bash
# Upgrade with new values
helm upgrade myapp ./myapp \
  --set replicaCount=4 \
  --set image.tag=1.26-alpine

# Upgrade with reuse of values
helm upgrade myapp ./myapp \
  --reuse-values \
  --set service.type=LoadBalancer

# Upgrade and wait for completion
helm upgrade myapp ./myapp --wait --timeout 5m

# Force upgrade (recreate resources)
helm upgrade myapp ./myapp --force
```

### 15. Rollback a Release
```bash
# View release history
helm history myapp

# Rollback to previous version
helm rollback myapp

# Rollback to specific revision
helm rollback myapp 2

# Rollback and wait
helm rollback myapp --wait
```

### 16. Add Chart Dependencies
Edit `myapp/Chart.yaml`:

```yaml
dependencies:
  - name: redis
    version: "17.11.3"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
  - name: postgresql
    version: "12.5.8"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

Update `values.yaml`:

```yaml
redis:
  enabled: true
  auth:
    enabled: true
    password: "redispassword"
  master:
    persistence:
      enabled: true
      size: 8Gi

postgresql:
  enabled: true
  auth:
    username: myapp
    password: "postgrespassword"
    database: myappdb
  primary:
    persistence:
      enabled: true
      size: 10Gi
```

Download dependencies:
```bash
# Update dependencies
helm dependency update myapp/

# List dependencies
helm dependency list myapp/

# Build dependencies
helm dependency build myapp/
```

### 17. Use Helm Hooks
Create `myapp/templates/job-migrate.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migrate
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    metadata:
      name: {{ include "myapp.fullname" . }}-migrate
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command:
        - sh
        - -c
        - echo "Running database migrations..."
```

### 18. Create Helm Tests
Create `myapp/templates/tests/test-connection.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "myapp.fullname" . }}-test-connection"
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "myapp.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

Run tests:
```bash
# Run Helm tests
helm test myapp

# View test logs
kubectl logs myapp-test-connection
```

### 19. Create a Helm Repository
```bash
# Create repository directory
mkdir helm-repo

# Copy packaged charts
cp myapp-1.0.0.tgz helm-repo/

# Generate index
helm repo index helm-repo/ --url https://your-domain.com/charts

# Upload to web server or Azure Blob Storage
az storage blob upload-batch \
  --account-name <storage-account> \
  --destination charts \
  --source helm-repo/

# Add your repository
helm repo add myrepo https://your-domain.com/charts
helm repo update
```

### 20. Advanced Helm Features

**Using Named Templates:**

Edit `myapp/templates/_helpers.tpl`:

```yaml
{{/*
Generate database URL
*/}}
{{- define "myapp.databaseURL" -}}
{{- if .Values.postgresql.enabled -}}
postgresql://{{ .Values.postgresql.auth.username }}:{{ .Values.postgresql.auth.password }}@{{ include "myapp.fullname" . }}-postgresql:5432/{{ .Values.postgresql.auth.database }}
{{- else -}}
{{ .Values.externalDatabase.url }}
{{- end -}}
{{- end -}}

{{/*
Generate Redis URL
*/}}
{{- define "myapp.redisURL" -}}
{{- if .Values.redis.enabled -}}
redis://:{{ .Values.redis.auth.password }}@{{ include "myapp.fullname" . }}-redis-master:6379
{{- else -}}
{{ .Values.externalRedis.url }}
{{- end -}}
{{- end -}}
```

**Using Values with Flow Control:**

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "myapp.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

**Uninstall Release:**
```bash
# Uninstall release
helm uninstall myapp

# Uninstall and keep history
helm uninstall myapp --keep-history

# Purge all data
helm uninstall myapp --no-hooks
```

## Expected Results
- Helm 3 installed and configured
- Custom Helm chart created with proper structure
- Chart packaged and validated
- Application deployed using Helm
- Dependencies managed automatically
- Releases upgraded and rolled back successfully
- Helm tests passing
- Chart repository created and accessible

## Key Takeaways
- **Helm** is a package manager for Kubernetes
- **Charts** are reusable Kubernetes application packages
- **Values** provide configuration flexibility
- **Templates** use Go templating for dynamic manifests
- **Releases** are installed instances of charts
- **Repositories** host and distribute charts
- **Hooks** enable lifecycle management (pre-install, post-upgrade, etc.)
- **Dependencies** allow chart composition
- `helm upgrade` enables rolling updates
- `helm rollback` provides easy recovery

## Helm Commands Quick Reference

| Command | Purpose |
|---------|---------|
| `helm install` | Install a chart |
| `helm upgrade` | Upgrade a release |
| `helm rollback` | Rollback to previous version |
| `helm uninstall` | Remove a release |
| `helm list` | List releases |
| `helm status` | Show release status |
| `helm get values` | Get release values |
| `helm template` | Render templates locally |
| `helm lint` | Validate chart |
| `helm package` | Package chart |
| `helm repo add` | Add repository |
| `helm dependency` | Manage dependencies |

## Troubleshooting
- **Chart validation fails**: Run `helm lint` for detailed errors
- **Template rendering errors**: Use `helm template --debug`
- **Upgrade fails**: Check `helm history` and rollback if needed
- **Dependencies not found**: Run `helm dependency update`
- **Values not applied**: Verify with `helm get values`

---


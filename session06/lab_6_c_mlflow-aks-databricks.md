# Lab 6c: MLflow on AKS with Databricks

## Objective
Deploy and manage ML models using MLflow on AKS integrated with Databricks.

## Prerequisites
- AKS cluster running
- Azure Databricks workspace
- Azure Container Registry (ACR)
- Azure ML workspace (optional)
- `kubectl` and `az` CLI configured

## Steps

### 1. Set Up Azure Container Registry
```bash
# Set variables
RESOURCE_GROUP="aks-databricks-rg"
ACR_NAME="aksmlflowacr$RANDOM"
LOCATION="eastus"
AKS_CLUSTER="aks-spark-cluster"

# Create ACR
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Standard \
  --location $LOCATION

# Enable admin access
az acr update \
  --name $ACR_NAME \
  --admin-enabled true

# Get ACR credentials
ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query username -o tsv)
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query passwords[0].value -o tsv)
ACR_SERVER="${ACR_NAME}.azurecr.io"

echo "ACR Server: $ACR_SERVER"
```

### 2. Attach ACR to AKS
```bash
# Attach ACR to AKS cluster
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER \
  --attach-acr $ACR_NAME

# Verify integration
az aks check-acr \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER \
  --acr "${ACR_SERVER}"
```

### 3. Deploy MLflow Tracking Server on AKS
Create namespace and storage:

```bash
# Create namespace
kubectl create namespace mlflow

# Create PVC for MLflow artifacts
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mlflow-pvc
  namespace: mlflow
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: default
  resources:
    requests:
      storage: 50Gi
EOF
```

Create PostgreSQL for MLflow backend:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-postgres
  namespace: mlflow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow-postgres
  template:
    metadata:
      labels:
        app: mlflow-postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: mlflow
        - name: POSTGRES_USER
          value: mlflow
        - name: POSTGRES_PASSWORD
          value: mlflow123
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: mlflow-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow-postgres
  namespace: mlflow
spec:
  selector:
    app: mlflow-postgres
  ports:
  - port: 5432
    targetPort: 5432
```

Deploy MLflow tracking server:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow-server
  namespace: mlflow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow-server
  template:
    metadata:
      labels:
        app: mlflow-server
    spec:
      containers:
      - name: mlflow
        image: ghcr.io/mlflow/mlflow:v2.9.0
        command:
        - mlflow
        - server
        - --backend-store-uri
        - postgresql://mlflow:mlflow123@mlflow-postgres:5432/mlflow
        - --default-artifact-root
        - /mlflow/artifacts
        - --host
        - 0.0.0.0
        - --port
        - "5000"
        ports:
        - containerPort: 5000
          name: http
        volumeMounts:
        - name: artifacts
          mountPath: /mlflow/artifacts
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
      volumes:
      - name: artifacts
        persistentVolumeClaim:
          claimName: mlflow-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow-server
  namespace: mlflow
spec:
  type: LoadBalancer
  selector:
    app: mlflow-server
  ports:
  - port: 5000
    targetPort: 5000
    name: http
```

Apply all:
```bash
kubectl apply -f mlflow-postgres.yaml
kubectl apply -f mlflow-server.yaml

# Wait for services to be ready
kubectl wait --for=condition=ready pod -l app=mlflow-postgres -n mlflow --timeout=300s
kubectl wait --for=condition=ready pod -l app=mlflow-server -n mlflow --timeout=300s

# Get MLflow UI URL
MLFLOW_URL=$(kubectl get svc mlflow-server -n mlflow -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "MLflow UI: http://$MLFLOW_URL:5000"
```

### 4. Configure Databricks to Use MLflow on AKS
Create notebook in Databricks:

```python
import mlflow
import os

# Set tracking URI to AKS MLflow server
mlflow_tracking_uri = "http://<MLFLOW_URL>:5000"
mlflow.set_tracking_uri(mlflow_tracking_uri)

# Verify connection
print(f"MLflow Tracking URI: {mlflow.get_tracking_uri()}")
print(f"MLflow Version: {mlflow.__version__}")
```

### 5. Train ML Model with MLflow Tracking
```python
import mlflow
import mlflow.sklearn
from sklearn.datasets import load_diabetes
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, r2_score
import numpy as np

# Set experiment
mlflow.set_experiment("diabetes-prediction")

# Load data
diabetes = load_diabetes()
X_train, X_test, y_train, y_test = train_test_split(
    diabetes.data, diabetes.target, test_size=0.2, random_state=42
)

# Start MLflow run
with mlflow.start_run(run_name="random-forest-v1"):
    # Log parameters
    n_estimators = 100
    max_depth = 10
    mlflow.log_param("n_estimators", n_estimators)
    mlflow.log_param("max_depth", max_depth)
    
    # Train model
    rf = RandomForestRegressor(
        n_estimators=n_estimators,
        max_depth=max_depth,
        random_state=42
    )
    rf.fit(X_train, y_train)
    
    # Make predictions
    predictions = rf.predict(X_test)
    
    # Log metrics
    mse = mean_squared_error(y_test, predictions)
    rmse = np.sqrt(mse)
    r2 = r2_score(y_test, predictions)
    
    mlflow.log_metric("mse", mse)
    mlflow.log_metric("rmse", rmse)
    mlflow.log_metric("r2_score", r2)
    
    # Log model
    mlflow.sklearn.log_model(
        rf,
        "model",
        registered_model_name="diabetes-rf-model"
    )
    
    # Log feature importances
    import matplotlib.pyplot as plt
    feature_importance = rf.feature_importances_
    plt.figure(figsize=(10, 6))
    plt.bar(range(len(feature_importance)), feature_importance)
    plt.xlabel("Feature Index")
    plt.ylabel("Importance")
    plt.title("Feature Importances")
    plt.tight_layout()
    plt.savefig("feature_importance.png")
    mlflow.log_artifact("feature_importance.png")
    
    print(f"MSE: {mse:.2f}")
    print(f"RMSE: {rmse:.2f}")
    print(f"R2 Score: {r2:.2f}")
```

### 6. Register Model in MLflow Model Registry
```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Get the best run
experiment = mlflow.get_experiment_by_name("diabetes-prediction")
runs = mlflow.search_runs(
    experiment_ids=[experiment.experiment_id],
    order_by=["metrics.rmse ASC"],
    max_results=1
)

best_run_id = runs.iloc[0]['run_id']
best_rmse = runs.iloc[0]['metrics.rmse']

print(f"Best Run ID: {best_run_id}")
print(f"Best RMSE: {best_rmse}")

# Register model
model_uri = f"runs:/{best_run_id}/model"
model_name = "diabetes-rf-model"

# Create registered model if it doesn't exist
try:
    client.create_registered_model(model_name)
except:
    pass

# Create model version
model_version = mlflow.register_model(model_uri, model_name)
print(f"Model Version: {model_version.version}")
```

### 7. Transition Model to Production
```python
# Transition model to production
client.transition_model_version_stage(
    name=model_name,
    version=model_version.version,
    stage="Production",
    archive_existing_versions=True
)

# Add description and tags
client.update_model_version(
    name=model_name,
    version=model_version.version,
    description="Random Forest model for diabetes prediction"
)

client.set_model_version_tag(
    name=model_name,
    version=model_version.version,
    key="team",
    value="data-science"
)

# Verify
latest_versions = client.get_latest_versions(model_name, stages=["Production"])
for version in latest_versions:
    print(f"Production Model Version: {version.version}")
```

### 8. Build Custom Docker Image for Model Serving
Create Dockerfile:

```dockerfile
FROM python:3.9-slim

# Install MLflow and dependencies
RUN pip install mlflow==2.9.0 scikit-learn pandas numpy

# Set environment variables
ENV MLFLOW_TRACKING_URI=http://mlflow-server.mlflow:5000

# Create app directory
WORKDIR /app

# Copy model serving script
COPY serve_model.py /app/

# Expose port
EXPOSE 8080

# Run model server
CMD ["python", "serve_model.py"]
```

Create model serving script `serve_model.py`:

```python
import mlflow
import mlflow.sklearn
from flask import Flask, request, jsonify
import pandas as pd
import os

app = Flask(__name__)

# Load model from MLflow
mlflow.set_tracking_uri(os.environ.get('MLFLOW_TRACKING_URI'))
model_name = "diabetes-rf-model"
model_stage = "Production"

model = mlflow.sklearn.load_model(f"models:/{model_name}/{model_stage}")
print(f"Model {model_name} ({model_stage}) loaded successfully")

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'healthy'}), 200

@app.route('/predict', methods=['POST'])
def predict():
    try:
        data = request.get_json()
        df = pd.DataFrame(data['instances'])
        predictions = model.predict(df)
        return jsonify({
            'predictions': predictions.tolist()
        })
    except Exception as e:
        return jsonify({'error': str(e)}), 400

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

Build and push to ACR:
```bash
# Build image
docker build -t ${ACR_SERVER}/mlflow-model-server:v1 .

# Login to ACR
az acr login --name $ACR_NAME

# Push image
docker push ${ACR_SERVER}/mlflow-model-server:v1
```

### 9. Deploy Model as Kubernetes Service
Create deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: diabetes-model-server
  namespace: mlflow
spec:
  replicas: 3
  selector:
    matchLabels:
      app: diabetes-model-server
  template:
    metadata:
      labels:
        app: diabetes-model-server
        version: v1
    spec:
      containers:
      - name: model-server
        image: <ACR_SERVER>/mlflow-model-server:v1
        env:
        - name: MLFLOW_TRACKING_URI
          value: http://mlflow-server.mlflow:5000
        ports:
        - containerPort: 8080
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: diabetes-model-server
  namespace: mlflow
spec:
  type: LoadBalancer
  selector:
    app: diabetes-model-server
  ports:
  - port: 80
    targetPort: 8080
    name: http
```

Deploy:
```bash
kubectl apply -f model-server-deployment.yaml

# Get service URL
MODEL_URL=$(kubectl get svc diabetes-model-server -n mlflow -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Model Server URL: http://$MODEL_URL"
```

### 10. Test Model Endpoint
```bash
# Test health endpoint
curl http://$MODEL_URL/health

# Test prediction endpoint
curl -X POST http://$MODEL_URL/predict \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [
      [0.03807591, 0.05068012, 0.06169621, 0.02187235, -0.0442235, -0.03482076, -0.04340085, -0.00259226, 0.01990842, -0.01764613]
    ]
  }'
```

From Databricks:
```python
import requests
import json

model_url = f"http://{MODEL_URL}/predict"

# Sample input
payload = {
    "instances": [
        [0.03807591, 0.05068012, 0.06169621, 0.02187235, -0.0442235, 
         -0.03482076, -0.04340085, -0.00259226, 0.01990842, -0.01764613]
    ]
}

# Make prediction
response = requests.post(model_url, json=payload)
predictions = response.json()

print("Predictions:", predictions)
```

### 11. Implement A/B Testing with Multiple Models
Train a second model variant:

```python
# Train alternative model (Gradient Boosting)
from sklearn.ensemble import GradientBoostingRegressor

with mlflow.start_run(run_name="gradient-boosting-v1"):
    # Log parameters
    n_estimators = 150
    learning_rate = 0.1
    mlflow.log_param("n_estimators", n_estimators)
    mlflow.log_param("learning_rate", learning_rate)
    
    # Train model
    gb = GradientBoostingRegressor(
        n_estimators=n_estimators,
        learning_rate=learning_rate,
        random_state=42
    )
    gb.fit(X_train, y_train)
    
    # Evaluate
    predictions = gb.predict(X_test)
    mse = mean_squared_error(y_test, predictions)
    rmse = np.sqrt(mse)
    r2 = r2_score(y_test, predictions)
    
    mlflow.log_metric("mse", mse)
    mlflow.log_metric("rmse", rmse)
    mlflow.log_metric("r2_score", r2)
    
    # Log model
    mlflow.sklearn.log_model(
        gb,
        "model",
        registered_model_name="diabetes-gb-model"
    )
```

Deploy both models with Istio traffic splitting:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: diabetes-model-v2
  namespace: mlflow
spec:
  replicas: 2
  selector:
    matchLabels:
      app: diabetes-model-server
      version: v2
  template:
    metadata:
      labels:
        app: diabetes-model-server
        version: v2
    spec:
      containers:
      - name: model-server
        image: <ACR_SERVER>/mlflow-model-gb:v1
        env:
        - name: MLFLOW_TRACKING_URI
          value: http://mlflow-server.mlflow:5000
        - name: MODEL_NAME
          value: diabetes-gb-model
        ports:
        - containerPort: 8080
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: diabetes-model
  namespace: mlflow
spec:
  hosts:
  - diabetes-model-server
  http:
  - match:
    - headers:
        x-model-version:
          exact: v2
    route:
    - destination:
        host: diabetes-model-server
        subset: v2
  - route:
    - destination:
        host: diabetes-model-server
        subset: v1
      weight: 90
    - destination:
        host: diabetes-model-server
        subset: v2
      weight: 10
```

### 12. Monitor Model Performance
Create monitoring script:

```python
import mlflow
from mlflow.tracking import MlflowClient
import time
from datetime import datetime

client = MlflowClient()

def log_prediction_metrics(model_name, predictions, actuals):
    """Log real-time prediction metrics"""
    with mlflow.start_run(run_name=f"monitoring-{datetime.now().isoformat()}"):
        mlflow.set_tag("monitoring", "true")
        mlflow.set_tag("model_name", model_name)
        
        # Calculate metrics
        from sklearn.metrics import mean_squared_error, r2_score
        import numpy as np
        
        mse = mean_squared_error(actuals, predictions)
        rmse = np.sqrt(mse)
        r2 = r2_score(actuals, predictions)
        
        mlflow.log_metric("mse", mse)
        mlflow.log_metric("rmse", rmse)
        mlflow.log_metric("r2_score", r2)
        mlflow.log_metric("prediction_count", len(predictions))
        
        print(f"Logged metrics - RMSE: {rmse:.2f}, R2: {r2:.2f}")

# Example usage
predictions = [150, 200, 175]
actuals = [155, 195, 180]
log_prediction_metrics("diabetes-rf-model", predictions, actuals)
```

### 13. Implement Model Retraining Pipeline
Create retraining job:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: model-retrain
  namespace: mlflow
spec:
  schedule: "0 2 * * 0"  # Every Sunday at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: retrain
            image: <ACR_SERVER>/mlflow-retrain:v1
            env:
            - name: MLFLOW_TRACKING_URI
              value: http://mlflow-server.mlflow:5000
            - name: DATA_SOURCE
              value: abfss://data@storage.dfs.core.windows.net/training/
          restartPolicy: OnFailure
```

Retraining script:

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestRegressor
import pandas as pd

def retrain_model():
    mlflow.set_experiment("diabetes-prediction-retrain")
    
    # Load new training data
    data = pd.read_csv(os.environ['DATA_SOURCE'])
    
    # Split features and target
    X = data.drop('target', axis=1)
    y = data['target']
    
    with mlflow.start_run():
        # Train model with updated data
        rf = RandomForestRegressor(n_estimators=100, random_state=42)
        rf.fit(X, y)
        
        # Log model
        mlflow.sklearn.log_model(rf, "model")
        
        # Evaluate and compare with production model
        # ... evaluation logic ...
        
        # If better, register new version
        mlflow.register_model("runs:/<run_id>/model", "diabetes-rf-model")

if __name__ == "__main__":
    retrain_model()
```

### 14. Feature Store Integration
Create feature store table:

```python
from databricks import feature_store
from pyspark.sql.functions import col, current_timestamp

# Initialize feature store
fs = feature_store.FeatureStoreClient()

# Create feature table
features_df = spark.createDataFrame([
    (1, 0.038, 0.050, 0.061, 0.021),
    (2, -0.001, -0.044, -0.051, -0.026),
], ["patient_id", "age", "sex", "bmi", "bp"])

fs.create_table(
    name="mlflow.patient_features",
    primary_keys=["patient_id"],
    df=features_df,
    description="Patient demographic features"
)

# Log model with feature store
from databricks.feature_store import FeatureFunction, FeatureLookup

feature_lookups = [
    FeatureLookup(
        table_name="mlflow.patient_features",
        lookup_key="patient_id"
    )
]

# Train and log model with features
with mlflow.start_run():
    # ... training code ...
    fs.log_model(
        model=rf,
        artifact_path="model",
        flavor=mlflow.sklearn,
        training_set=training_set,
        registered_model_name="diabetes-model-with-features"
    )
```

### 15. Model Explainability with SHAP
```python
import shap
import mlflow

# Load production model
model = mlflow.sklearn.load_model("models:/diabetes-rf-model/Production")

# Generate SHAP values
explainer = shap.TreeExplainer(model)
shap_values = explainer.shap_values(X_test)

# Log SHAP summary plot
shap.summary_plot(shap_values, X_test, show=False)
import matplotlib.pyplot as plt
plt.tight_layout()
mlflow.log_figure(plt.gcf(), "shap_summary.png")

# Log SHAP values as artifact
import numpy as np
np.save("shap_values.npy", shap_values)
mlflow.log_artifact("shap_values.npy")
```

### 16. Create MLflow Project
Create `MLproject` file:

```yaml
name: diabetes-prediction

conda_env: conda.yaml

entry_points:
  main:
    parameters:
      n_estimators: {type: int, default: 100}
      max_depth: {type: int, default: 10}
    command: "python train.py --n-estimators {n_estimators} --max-depth {max_depth}"
  
  evaluate:
    parameters:
      model_uri: {type: string}
    command: "python evaluate.py --model-uri {model_uri}"
```

Run project:
```python
mlflow.run(
    ".",
    parameters={"n_estimators": 150, "max_depth": 15}
)
```

### 17. Set Up Model Alerts
Create alert configuration:

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Set alert thresholds
threshold_rmse = 60.0

def check_model_performance():
    # Get latest production model metrics
    latest_run = client.search_runs(
        experiment_ids=["1"],
        filter_string="tags.monitoring = 'true'",
        order_by=["start_time DESC"],
        max_results=1
    )[0]
    
    rmse = latest_run.data.metrics.get('rmse', 0)
    
    if rmse > threshold_rmse:
        # Send alert (integrate with Azure Monitor, email, etc.)
        print(f"ALERT: Model RMSE {rmse} exceeds threshold {threshold_rmse}")
        # trigger_alert(rmse)
    
    return rmse

# Schedule this function to run periodically
current_rmse = check_model_performance()
print(f"Current RMSE: {current_rmse}")
```

### 18. Cleanup
```bash
# Delete Kubernetes resources
kubectl delete namespace mlflow

# Delete ACR images
az acr repository delete \
  --name $ACR_NAME \
  --repository mlflow-model-server \
  --yes

# Delete ACR (optional)
# az acr delete --name $ACR_NAME --resource-group $RESOURCE_GROUP --yes
```

## Expected Results
- MLflow tracking server running on AKS
- Models trained from Databricks tracked in MLflow
- Model registry managing model versions
- Models deployed as Kubernetes services
- Real-time predictions via REST API
- A/B testing between model variants
- Model monitoring and retraining pipelines
- Feature store integration
- Model explainability with SHAP

## Key Takeaways
- **MLflow** provides end-to-end ML lifecycle management
- **Tracking server** logs experiments, parameters, metrics
- **Model registry** manages model versions and stages
- **Kubernetes** provides scalable model serving
- **A/B testing** compares model variants in production
- **Monitoring** tracks model performance over time
- **Retraining** keeps models current with new data
- **Feature store** ensures consistent features
- **Explainability** builds trust in model predictions
- Integration between Databricks, MLflow, and AKS

## MLflow Components

| Component | Purpose |
|-----------|---------|
| Tracking | Log experiments and metrics |
| Projects | Reproducible runs |
| Models | Package models in standard format |
| Registry | Manage model lifecycle |
| Deployments | Serve models |

## Model Stages

| Stage | Purpose |
|-------|---------|
| None | Initial state |
| Staging | Testing in pre-prod |
| Production | Serving live traffic |
| Archived | Retired models |

## Troubleshooting
- **Model server not starting**: Check MLflow tracking URI
- **Predictions failing**: Verify input data format
- **High latency**: Scale deployment replicas
- **Version conflicts**: Pin dependency versions
- **Storage issues**: Increase PVC size

---


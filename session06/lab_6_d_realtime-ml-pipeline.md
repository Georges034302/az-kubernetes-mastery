# Lab 6d: Real-Time ML Inference Pipeline
<img width="1488" height="608" alt="IMG" src="https://github.com/user-attachments/assets/fdfbf5a1-72ae-4cf9-bf15-db0ab3e0d364" />

## Objective
Build a real-time ML inference pipeline on AKS using Event Hubs (Kafka-compatible), Redis caching, and streaming frameworks. Deploy scalable inference services with monitoring, auto-scaling, and resilience patterns.

---

## Lab Parameters

Set these variables at the start:

```bash
# Azure Resources
RESOURCE_GROUP="rg-aks-ml-pipeline"
LOCATION="australiaeast"
CLUSTER_NAME="aks-inference-cluster"
NODE_COUNT=3
NODE_SIZE="Standard_D4s_v3"
K8S_VERSION="1.28"

# Azure Container Registry
ACR_NAME="mlpipelineacr$RANDOM"
ACR_SKU="Standard"

# Event Hubs Configuration
EVENTHUB_NAMESPACE="evhns-ml-inference$RANDOM"
EVENTHUB_NAME="inference-requests"
EVENTHUB_RESULTS="inference-results"
EVENTHUB_SKU="Standard"
EVENTHUB_PARTITIONS=4
CONSUMER_GROUP="inference-consumer"

# MLflow Configuration  
MLFLOW_NAMESPACE="mlflow"
MODEL_NAME="diabetes-rf-model"
MODEL_STAGE="Production"

# Inference Service
INFERENCE_NAMESPACE="inference"
INFERENCE_REPLICAS=3
```

---

## Step 1: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \      # `Resource group name`
  --location $LOCATION          # `Azure region (Sydney, Australia)`
```

---

## Step 2: Create Azure Container Registry

```bash
az acr create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --name $ACR_NAME \                       # `ACR name`
  --sku $ACR_SKU \                         # `SKU tier`
  --location $LOCATION \                   # `Region`
  --admin-enabled true                     # `Enable admin account`

ACR_USERNAME=$(az acr credential show \
  --name $ACR_NAME \
  --query username \
  -o tsv)                                  # `Get ACR username`

ACR_PASSWORD=$(az acr credential show \
  --name $ACR_NAME \
  --query passwords[0].value \
  -o tsv)                                  # `Get ACR password`

ACR_SERVER="${ACR_NAME}.azurecr.io"        # `Construct ACR server URL`

echo "ACR created: $ACR_NAME"
echo "ACR Server: $ACR_SERVER"
```

---

## Step 3: Create AKS Cluster

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --name $CLUSTER_NAME \                   # `Cluster name`
  --location $LOCATION \                   # `Region`
  --node-count $NODE_COUNT \               # `Number of nodes`
  --node-vm-size $NODE_SIZE \              # `VM size (4 vCPU, 16GB for ML)`
  --kubernetes-version $K8S_VERSION \      # `Kubernetes version`
  --enable-managed-identity \              # `Use managed identity`
  --generate-ssh-keys \                    # `Generate SSH keys`
  --network-plugin azure \                 # `Azure CNI networking`
  --attach-acr $ACR_NAME \                 # `Attach ACR for image pull`
  --no-wait                                # `Don't wait for completion`

az aks wait \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --created                                # `Wait for cluster creation`

az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing                     # `Get cluster credentials`

echo "AKS cluster created: $CLUSTER_NAME"
```

---

## Step 4: Create Azure Event Hubs Namespace

```bash
az eventhubs namespace create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --name $EVENTHUB_NAMESPACE \             # `Event Hubs namespace`
  --location $LOCATION \                   # `Region`
  --sku $EVENTHUB_SKU \                    # `SKU (Standard for Kafka)`
  --enable-kafka true                      # `Enable Kafka protocol`

az eventhubs namespace wait \
  --resource-group $RESOURCE_GROUP \
  --name $EVENTHUB_NAMESPACE \
  --created                                # `Wait for namespace creation`

echo "Event Hubs namespace created: $EVENTHUB_NAMESPACE"
```

---

## Step 5: Create Event Hubs

Create Event Hub for requests:
```bash
az eventhubs eventhub create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --namespace-name $EVENTHUB_NAMESPACE \   # `Namespace`
  --name $EVENTHUB_NAME \                  # `Event Hub name for requests`
  --partition-count $EVENTHUB_PARTITIONS \ # `Number of partitions`
  --message-retention 1                    # `Retention in days`

echo "Event Hub created for requests: $EVENTHUB_NAME"
```

Create Event Hub for results:
```bash
az eventhubs eventhub create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --namespace-name $EVENTHUB_NAMESPACE \   # `Namespace`
  --name $EVENTHUB_RESULTS \               # `Event Hub name for results`
  --partition-count $EVENTHUB_PARTITIONS \ # `Number of partitions`
  --message-retention 1                    # `Retention in days`

echo "Event Hub created for results: $EVENTHUB_RESULTS"
```

Create consumer group:
```bash
az eventhubs eventhub consumer-group create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --namespace-name $EVENTHUB_NAMESPACE \   # `Namespace`
  --eventhub-name $EVENTHUB_NAME \         # `Event Hub name`
  --name $CONSUMER_GROUP                   # `Consumer group name`

echo "Consumer group created: $CONSUMER_GROUP"
```

Get connection string:
```bash
EVENTHUB_CONNECTION=$(az eventhubs namespace authorization-rule keys list \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $EVENTHUB_NAMESPACE \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString \
  -o tsv)                                  # `Get connection string`

echo "Event Hubs connection string retrieved (${#EVENTHUB_CONNECTION} chars)"
```

---

## Step 6: Deploy Redis Cache

Install Redis for prediction caching:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami  # `Add Bitnami Helm repo`
helm repo update                                           # `Update Helm repos`

kubectl create namespace $INFERENCE_NAMESPACE              # `Create inference namespace`

helm install redis bitnami/redis \
  --namespace $INFERENCE_NAMESPACE \                       # `Namespace`
  --set auth.enabled=false \                               # `Disable auth for simplicity`
  --set master.persistence.size=10Gi \                     # `Storage size`
  --set replica.replicaCount=2                             # `Number of replicas`

kubectl wait \
  --for=condition=ready pod \
  -l app.kubernetes.io/name=redis \
  --namespace $INFERENCE_NAMESPACE \
  --timeout=300s                                           # `Wait for Redis to be ready`

REDIS_HOST=$(kubectl get svc redis-master \
  --namespace $INFERENCE_NAMESPACE \
  -o jsonpath='{.spec.clusterIP}')                         # `Get Redis internal IP`

echo "Redis deployed"
echo "Redis Host: $REDIS_HOST"
```

---

## Step 7: Create Kubernetes Secrets

Create secrets for Event Hubs:
```bash
kubectl create secret generic eventhub-secret \
  --from-literal=connection-string="$EVENTHUB_CONNECTION" \
  --namespace $INFERENCE_NAMESPACE                         # `Event Hub connection secret`

kubectl create secret generic redis-config \
  --from-literal=host="$REDIS_HOST" \
  --from-literal=port="6379" \
  --namespace $INFERENCE_NAMESPACE                         # `Redis configuration secret`

echo "Secrets created in namespace: $INFERENCE_NAMESPACE"
```

---

## Step 8: Create Real-Time Inference Service

Create `inference-service.py`:

```python
import os
import json
import mlflow
import mlflow.sklearn
from kafka import KafkaConsumer, KafkaProducer
from azure.eventhub import EventHubConsumerClient, EventHubProducerClient
from azure.eventhub import EventData
import numpy as np
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class InferenceService:
    def __init__(self, use_eventhub=True):
        # Load MLflow model
        mlflow.set_tracking_uri(os.environ.get('MLFLOW_TRACKING_URI'))
        model_name = os.environ.get('MODEL_NAME', 'diabetes-rf-model')
        model_stage = os.environ.get('MODEL_STAGE', 'Production')
        
        logger.info(f"Loading model {model_name}/{model_stage}")
        self.model = mlflow.sklearn.load_model(f"models:/{model_name}/{model_stage}")
        logger.info("Model loaded successfully")
        
        self.use_eventhub = use_eventhub
        
        if use_eventhub:
            self.setup_eventhub()
        else:
            self.setup_kafka()
    
    def setup_eventhub(self):
        """Setup Azure Event Hub connections"""
        connection_string = os.environ['EVENTHUB_CONNECTION']
        eventhub_name = os.environ.get('EVENTHUB_NAME', 'inference-requests')
        consumer_group = os.environ.get('CONSUMER_GROUP', '$Default')
        
        self.consumer = EventHubConsumerClient.from_connection_string(
            connection_string,
            consumer_group=consumer_group,
            eventhub_name=eventhub_name
        )
        
        self.producer = EventHubProducerClient.from_connection_string(
            connection_string,
            eventhub_name=f"{eventhub_name}-results"
        )
        
        logger.info("Event Hub configured")
    
    def setup_kafka(self):
        """Setup Kafka connections"""
        kafka_broker = os.environ.get('KAFKA_BROKER', 'kafka.kafka:9092')
        
        self.consumer = KafkaConsumer(
            'inference-requests',
            bootstrap_servers=[kafka_broker],
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='latest',
            enable_auto_commit=True,
            group_id='inference-service'
        )
        
        self.producer = KafkaProducer(
            bootstrap_servers=[kafka_broker],
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
        
        logger.info("Kafka configured")
    
    def process_eventhub_event(self, partition_context, event):
        """Process Event Hub event"""
        try:
            data = json.loads(event.body_as_str())
            logger.info(f"Received request: {data}")
            
            # Make prediction
            features = np.array(data['features']).reshape(1, -1)
            prediction = self.model.predict(features)[0]
            
            # Send result
            result = {
                'request_id': data.get('request_id'),
                'prediction': float(prediction),
                'model_version': data.get('model_version'),
                'timestamp': event.enqueued_time.isoformat()
            }
            
            event_data = EventData(json.dumps(result))
            event_batch = self.producer.create_batch()
            event_batch.add(event_data)
            self.producer.send_batch(event_batch)
            
            logger.info(f"Sent prediction: {prediction}")
            
            partition_context.update_checkpoint(event)
        except Exception as e:
            logger.error(f"Error processing event: {e}")
    
    def process_kafka_message(self, message):
        """Process Kafka message"""
        try:
            data = message.value
            logger.info(f"Received request: {data}")
            
            # Make prediction
            features = np.array(data['features']).reshape(1, -1)
            prediction = self.model.predict(features)[0]
            
            # Send result
            result = {
                'request_id': data.get('request_id'),
                'prediction': float(prediction),
                'model_version': data.get('model_version'),
                'timestamp': message.timestamp
            }
            
            self.producer.send('inference-results', value=result)
            
            logger.info(f"Sent prediction: {prediction}")
        except Exception as e:
            logger.error(f"Error processing message: {e}")
    
    def run(self):
        """Start processing messages"""
        logger.info("Starting inference service...")
        
        if self.use_eventhub:
            with self.consumer:
                self.consumer.receive(
                    on_event=self.process_eventhub_event,
                    starting_position="-1"
                )
        else:
            for message in self.consumer:
                self.process_kafka_message(message)

if __name__ == "__main__":
    use_eventhub = os.environ.get('USE_EVENTHUB', 'true').lower() == 'true'
    service = InferenceService(use_eventhub=use_eventhub)
    service.run()
```

---

## Step 9: Build and Deploy Inference Service

Create Dockerfile:
```dockerfile
FROM python:3.9-slim

# Install dependencies
RUN pip install mlflow==2.9.0 scikit-learn pandas numpy \
    azure-eventhub redis prometheus-client

# Copy service code
COPY inference-service.py /app/

WORKDIR /app

CMD ["python", "inference-service.py"]
```

Build and push image:
```bash
docker build \
  -t ${ACR_SERVER}/inference-service:v1 \
  .                                        # `Build Docker image`

az acr login \
  --name $ACR_NAME                         # `Login to ACR`

docker push ${ACR_SERVER}/inference-service:v1  # `Push image to ACR`

echo "Inference service image pushed to ACR"
```

Deploy inference service to Kubernetes:
```bash
kubectl apply --namespace $INFERENCE_NAMESPACE -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-service
spec:
  replicas: $INFERENCE_REPLICAS
  selector:
    matchLabels:
      app: inference-service
  template:
    metadata:
      labels:
        app: inference-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
    spec:
      containers:
      - name: inference
        image: ${ACR_SERVER}/inference-service:v1
        env:
        - name: MLFLOW_TRACKING_URI
          value: http://mlflow-server.$MLFLOW_NAMESPACE:5000
        - name: MODEL_NAME
          value: $MODEL_NAME
        - name: MODEL_STAGE
          value: $MODEL_STAGE
        - name: USE_EVENTHUB
          value: "true"
        - name: EVENTHUB_CONNECTION
          valueFrom:
            secretKeyRef:
              name: eventhub-secret
              key: connection-string
        - name: EVENTHUB_NAME
          value: $EVENTHUB_NAME
        - name: EVENTHUB_RESULTS
          value: $EVENTHUB_RESULTS
        - name: CONSUMER_GROUP
          value: $CONSUMER_GROUP
        - name: REDIS_HOST
          valueFrom:
            secretKeyRef:
              name: redis-config
              key: host
        - name: REDIS_PORT
          valueFrom:
            secretKeyRef:
              name: redis-config
              key: port
        ports:
        - containerPort: 8000
          name: metrics
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
  name: inference-service
spec:
  selector:
    app: inference-service
  ports:
  - name: metrics
    port: 8000
    targetPort: 8000
EOF

kubectl wait \
  --for=condition=ready pod \
  -l app=inference-service \
  --namespace $INFERENCE_NAMESPACE \
  --timeout=300s                           # `Wait for inference service`

echo "Inference service deployed with $INFERENCE_REPLICAS replicas"
```

---

## Step 10: Create Request Producer in Databricks

Create a Databricks notebook to send inference requests:

```python
from azure.eventhub import EventHubProducerClient, EventData
import json
import time
import random

# Event Hub configuration
connection_string = dbutils.secrets.get(scope="eventhub", key="connection-string")
eventhub_name = "inference-requests"

# Create producer
producer = EventHubProducerClient.from_connection_string(
    connection_string,
    eventhub_name=eventhub_name
)

# Generate sample inference requests
def generate_request():
    return {
        'request_id': f"req_{int(time.time()*1000)}",
        'features': [random.uniform(-0.1, 0.1) for _ in range(10)],
        'model_version': 'v1'
    }

# Send requests
with producer:
    for i in range(100):
        request = generate_request()
        event_data = EventData(json.dumps(request))
        
        event_batch = producer.create_batch()
        event_batch.add(event_data)
        producer.send_batch(event_batch)
        
        print(f"Sent request {i}: {request['request_id']}")
        time.sleep(0.1)

print("Finished sending requests")
```

---

## Step 11: Consume Results in Databricks
```python
from azure.eventhub import EventHubConsumerClient

connection_string = dbutils.secrets.get(scope="eventhub", key="connection-string")
eventhub_name = "inference-requests-results"

def on_event(partition_context, event):
    result = json.loads(event.body_as_str())
    print(f"Prediction: {result}")
    partition_context.update_checkpoint(event)

consumer = EventHubConsumerClient.from_connection_string(
    connection_string,
    consumer_group="$Default",
    eventhub_name=eventhub_name
)

with consumer:
    consumer.receive(
        on_event=on_event,
        starting_position="-1"
    )
```

---

## Step 12: Implement Batch Inference with Spark Streaming

Create a Spark Structured Streaming job in Databricks:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json, to_json, struct
from pyspark.sql.types import StructType, StructField, StringType, ArrayType, DoubleType
import mlflow

# Initialize Spark
spark = SparkSession.builder \
    .appName("BatchInferencePipeline") \
    .getOrCreate()

# Event Hub configuration
eventhub_config = {
    "eventhubs.connectionString": sc._jvm.org.apache.spark.eventhubs.EventHubsUtils.encrypt(connection_string),
    "eventhubs.consumerGroup": "batch-inference",
    "maxEventsPerTrigger": 1000
}

# Define schema
request_schema = StructType([
    StructField("request_id", StringType(), True),
    StructField("features", ArrayType(DoubleType()), True),
    StructField("model_version", StringType(), True)
])

# Read from Event Hub
df_requests = spark.readStream \
    .format("eventhubs") \
    .options(**eventhub_config) \
    .load()

# Parse JSON
df_parsed = df_requests.select(
    from_json(col("body").cast("string"), request_schema).alias("data")
).select("data.*")

# Load MLflow model as UDF
model_uri = "models:/diabetes-rf-model/Production"
predict_udf = mlflow.pyfunc.spark_udf(spark, model_uri, result_type="double")

# Apply predictions
df_predictions = df_parsed.withColumn(
    "prediction",
    predict_udf(struct(*[col("features")[i] for i in range(10)]))
)

# Prepare output
df_output = df_predictions.select(
    to_json(struct("request_id", "prediction", "model_version")).alias("body")
)

# Write to output Event Hub
output_config = eventhub_config.copy()
output_config["eventhubs.connectionString"] = sc._jvm.org.apache.spark.eventhubs.EventHubsUtils.encrypt(
    connection_string.replace("inference-requests", "inference-requests-results")
)

query = df_output.writeStream \
    .format("eventhubs") \
    .options(**output_config) \
    .option("checkpointLocation", "/mnt/checkpoints/batch-inference") \
    .start()

query.awaitTermination()
```

### 9. Add Request/Response Monitoring
Create monitoring deployment:

```python
from prometheus_client import Counter, Histogram, start_http_server
import time

# Metrics
request_count = Counter('inference_requests_total', 'Total inference requests')
prediction_latency = Histogram('inference_latency_seconds', 'Prediction latency')
error_count = Counter('inference_errors_total', 'Total inference errors')

def monitored_predict(features):
    request_count.inc()
    
    start_time = time.time()
    try:
        prediction = model.predict(features)
        prediction_latency.observe(time.time() - start_time)
        return prediction
    except Exception as e:
        error_count.inc()
        raise e

# Start Prometheus metrics server
start_http_server(8000)
```

---

## Additional Implementation Patterns

### Model Versioning in Pipeline
```python
class VersionedInferenceService:
    def __init__(self):
        self.models = {}
        self.load_models()
    
    def load_models(self):
        """Load multiple model versions"""
        versions = ['v1', 'v2', 'v3']
        for version in versions:
            try:
                model = mlflow.sklearn.load_model(f"models:/diabetes-rf-model/{version}")
                self.models[version] = model
                logger.info(f"Loaded model version {version}")
            except:
                logger.warning(f"Version {version} not available")
    
    def predict(self, features, version='Production'):
        """Make prediction with specific version"""
        if version not in self.models:
            # Fallback to production
            version = 'Production'
            if version not in self.models:
                raise ValueError("No models available")
        
        return self.models[version].predict(features)
```

### Feature Preprocessing Pipeline
```python
from sklearn.preprocessing import StandardScaler
import joblib

class PreprocessingPipeline:
    def __init__(self):
        # Load preprocessing artifacts from MLflow
        self.scaler = joblib.load('scaler.pkl')
        self.feature_names = ['age', 'sex', 'bmi', 'bp', 's1', 's2', 's3', 's4', 's5', 's6']
    
    def preprocess(self, raw_features):
        """Preprocess raw features"""
        # Handle missing values
        features = self.handle_missing(raw_features)
        
        # Scale features
        features_scaled = self.scaler.transform(features)
        
        # Validate feature range
        self.validate_features(features_scaled)
        
        return features_scaled
    
    def handle_missing(self, features):
        """Handle missing values"""
        # Implement missing value logic
        return features
    
    def validate_features(self, features):
        """Validate feature values"""
        if np.any(np.isnan(features)):
            raise ValueError("Features contain NaN values")
        if np.any(np.isinf(features)):
            raise ValueError("Features contain infinite values")
```

### Caching Layer Implementation

Redis caching is already deployed in Step 6. Add this caching logic to inference service:

```python
import redis
import hashlib
import json

class CachedInferenceService:
    def __init__(self):
        self.redis_client = redis.Redis(
            host=os.environ.get('REDIS_HOST', 'redis-master.mlflow'),
            port=6379,
            db=0
        )
        self.cache_ttl = 3600  # 1 hour
    
    def get_cache_key(self, features):
        """Generate cache key from features"""
        feature_str = json.dumps(features.tolist(), sort_keys=True)
        return hashlib.md5(feature_str.encode()).hexdigest()
    
    def predict_with_cache(self, features):
        """Predict with caching"""
        cache_key = self.get_cache_key(features)
        
        # Check cache
        cached_result = self.redis_client.get(cache_key)
        if cached_result:
            logger.info("Cache hit")
            return json.loads(cached_result)
        
        # Make prediction
        prediction = self.model.predict(features)
        
        # Cache result
        self.redis_client.setex(
            cache_key,
            self.cache_ttl,
            json.dumps(prediction.tolist())
        )
        
        return prediction
```

### Request Queue with Priority
```python
import heapq
from dataclasses import dataclass, field
from typing import Any

@dataclass(order=True)
class PrioritizedRequest:
    priority: int
    request: Any = field(compare=False)
    timestamp: float = field(compare=False)

class PriorityInferenceQueue:
    def __init__(self):
        self.queue = []
    
    def add_request(self, request, priority=1):
        """Add request with priority (lower number = higher priority)"""
        item = PrioritizedRequest(priority, request, time.time())
        heapq.heappush(self.queue, item)
    
    def process_next(self):
        """Process highest priority request"""
        if not self.queue:
            return None
        
        item = heapq.heappop(self.queue)
        
        # Check if request is too old
        if time.time() - item.timestamp > 60:  # 60 second timeout
            logger.warning(f"Request timed out: {item.request['request_id']}")
            return None
        
        return item.request
```

### Circuit Breaker Pattern
```python
from datetime import datetime, timedelta

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker"""
        if self.state == 'OPEN':
            if datetime.now() - self.last_failure_time > timedelta(seconds=self.timeout):
                self.state = 'HALF_OPEN'
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        """Handle successful call"""
        self.failures = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        """Handle failed call"""
        self.failures += 1
        self.last_failure_time = datetime.now()
        
        if self.failures >= self.failure_threshold:
            self.state = 'OPEN'
            logger.error("Circuit breaker opened")
```

---

## Step 13: Configure Horizontal Pod Autoscaler

Create HPA for auto-scaling:
```bash
kubectl apply --namespace $INFERENCE_NAMESPACE -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 5 minutes stabilization
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
EOF

echo "HPA configured: 3-20 replicas based on CPU (70%) and Memory (80%)"
```

---

## Step 14: Monitor with Prometheus ServiceMonitor

Create Prometheus ServiceMonitor for metrics collection:

```bash
kubectl apply --namespace $INFERENCE_NAMESPACE -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: inference-service-metrics
spec:
  selector:
    matchLabels:
      app: inference-service
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
EOF

echo "ServiceMonitor created for Prometheus scraping"
```

View metrics:
```bash
kubectl port-forward \
  --namespace $INFERENCE_NAMESPACE \
  --address 0.0.0.0 \
  svc/inference-service 8000:8000 &       # `Port-forward metrics endpoint`

echo "Metrics available at: http://localhost:8000/metrics"
```

---

## Step 15: Load Testing

Create load test script in Databricks:

```python
import concurrent.futures
import time
from azure.eventhub import EventHubProducerClient, EventData
import json
import random

def send_request(producer, request_id):
    """Send single inference request"""
    request = {
        'request_id': f"load_test_{request_id}",
        'features': [random.uniform(-0.1, 0.1) for _ in range(10)],
        'model_version': 'v1'
    }
    
    event_data = EventData(json.dumps(request))
    event_batch = producer.create_batch()
    event_batch.add(event_data)
    producer.send_batch(event_batch)

def load_test(num_requests=10000, num_workers=50):
    """Run load test"""
    producer = EventHubProducerClient.from_connection_string(
        connection_string,
        eventhub_name="inference-requests"
    )
    
    start_time = time.time()
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_workers) as executor:
        futures = [
            executor.submit(send_request, producer, i)
            for i in range(num_requests)
        ]
        concurrent.futures.wait(futures)
    
    duration = time.time() - start_time
    throughput = num_requests / duration
    
    print(f"Sent {num_requests} requests in {duration:.2f} seconds")
    print(f"Throughput: {throughput:.2f} requests/second")
    
    producer.close()

# Run load test
load_test(num_requests=10000, num_workers=100)
```

---

## Cleanup

### Option 1: Delete Inference Resources Only
```bash
kubectl delete namespace $INFERENCE_NAMESPACE  # `Delete inference namespace and all resources`

echo "Inference resources deleted"
```

### Option 2: Delete Event Hubs
```bash
az eventhubs namespace delete \
  --resource-group $RESOURCE_GROUP \
  --name $EVENTHUB_NAMESPACE \
  --yes                                        # `Delete Event Hubs namespace`

echo "Event Hubs namespace deleted"
```

### Option 3: Delete ACR Images
```bash
az acr repository delete \
  --name $ACR_NAME \
  --repository inference-service \
  --yes                                        # `Delete inference service image`

echo "ACR images deleted"
```

### Option 4: Delete All Azure Resources
```bash
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait                                    # `Delete entire resource group`

echo "Resource group deletion initiated: $RESOURCE_GROUP"
```

## Expected Results
- Real-time inference pipeline processing Event Hub/Kafka messages
- Spark Structured Streaming for batch inference
- Model predictions cached in Redis
- Horizontal pod autoscaling based on load
- Request prioritization and circuit breaking
- Prometheus metrics and Grafana dashboards
- Load testing achieving high throughput
- End-to-end latency under 100ms

## Key Takeaways
- **Event-driven architecture** enables real-time ML inference
- **Spark Structured Streaming** handles batch inference
- **Caching** improves latency and reduces compute costs
- **Circuit breakers** prevent cascading failures
- **Priority queues** handle different SLA requirements
- **HPA** scales based on actual demand
- **Monitoring** provides visibility into pipeline health
- **Load testing** validates performance requirements
- Integration of Databricks, Kafka, Redis, and Kubernetes

## Pipeline Components

| Component | Purpose |
|-----------|---------|
| Event Hub/Kafka | Message queue |
| Inference Service | Model serving |
| Redis | Prediction caching |
| Spark Streaming | Batch processing |
| Prometheus | Metrics collection |
| HPA | Auto-scaling |

## Performance Metrics

| Metric | Target |
|--------|--------|
| Latency (P95) | < 100ms |
| Throughput | > 1000 req/s |
| Error Rate | < 0.1% |
| Cache Hit Rate | > 70% |
| CPU Utilization | 60-80% |

## Troubleshooting
- **High latency**: Enable caching, increase replicas
- **Consumer lag**: Scale up inference service
- **Memory issues**: Optimize model size
- **Cache misses**: Review cache TTL and key strategy
- **Event Hub throttling**: Increase throughput units

---



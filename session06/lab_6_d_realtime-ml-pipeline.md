# Lab 6d: Real-Time ML Inference Pipeline

## Objective
Build a real-time ML inference pipeline on AKS using Databricks, Kafka, and streaming frameworks.

## Prerequisites
- AKS cluster running
- Azure Databricks workspace
- Azure Event Hubs (Kafka-compatible)
- MLflow model from Lab 6c
- `kubectl` configured

## Steps

### 1. Create Azure Event Hubs Namespace
```bash
# Set variables
RESOURCE_GROUP="aks-databricks-rg"
EVENTHUB_NAMESPACE="aks-ml-events$RANDOM"
EVENTHUB_NAME="inference-requests"
LOCATION="eastus"

# Create Event Hubs namespace
az eventhubs namespace create \
  --resource-group $RESOURCE_GROUP \
  --name $EVENTHUB_NAMESPACE \
  --location $LOCATION \
  --sku Standard

# Create Event Hub
az eventhubs eventhub create \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $EVENTHUB_NAMESPACE \
  --name $EVENTHUB_NAME \
  --partition-count 4 \
  --message-retention 1

# Create consumer group
az eventhubs eventhub consumer-group create \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $EVENTHUB_NAMESPACE \
  --eventhub-name $EVENTHUB_NAME \
  --name inference-consumer

# Get connection string
EVENTHUB_CONNECTION=$(az eventhubs namespace authorization-rule keys list \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $EVENTHUB_NAMESPACE \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv)

echo "Event Hub Connection String saved"
```

### 2. Deploy Kafka (Alternative to Event Hubs)
For on-premise Kafka on AKS:

```bash
# Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install Kafka
kubectl create namespace kafka

helm install kafka bitnami/kafka \
  --namespace kafka \
  --set replicaCount=3 \
  --set auth.clientProtocol=plaintext \
  --set persistence.size=20Gi

# Wait for Kafka to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kafka -n kafka --timeout=300s

# Get Kafka connection details
export KAFKA_BROKER=$(kubectl get svc kafka -n kafka -o jsonpath='{.spec.clusterIP}'):9092
echo "Kafka Broker: $KAFKA_BROKER"
```

### 3. Create Kubernetes Secret for Event Hub
```bash
# Create secret with Event Hub connection
kubectl create secret generic eventhub-secret \
  --from-literal=connection-string="$EVENTHUB_CONNECTION" \
  -n mlflow

# For Kafka
kubectl create secret generic kafka-secret \
  --from-literal=broker="$KAFKA_BROKER" \
  -n mlflow
```

### 4. Deploy Real-Time Inference Service
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

### 5. Build and Deploy Inference Service
Create Dockerfile:

```dockerfile
FROM python:3.9-slim

# Install dependencies
RUN pip install mlflow==2.9.0 scikit-learn pandas numpy \
    azure-eventhub kafka-python

# Copy service code
COPY inference-service.py /app/

WORKDIR /app

CMD ["python", "inference-service.py"]
```

Build and push:
```bash
ACR_NAME="<your-acr-name>"
ACR_SERVER="${ACR_NAME}.azurecr.io"

docker build -t ${ACR_SERVER}/inference-service:v1 .
az acr login --name $ACR_NAME
docker push ${ACR_SERVER}/inference-service:v1
```

Deploy to Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-service
  namespace: mlflow
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inference-service
  template:
    metadata:
      labels:
        app: inference-service
    spec:
      containers:
      - name: inference
        image: <ACR_SERVER>/inference-service:v1
        env:
        - name: MLFLOW_TRACKING_URI
          value: http://mlflow-server.mlflow:5000
        - name: MODEL_NAME
          value: diabetes-rf-model
        - name: MODEL_STAGE
          value: Production
        - name: USE_EVENTHUB
          value: "true"
        - name: EVENTHUB_CONNECTION
          valueFrom:
            secretKeyRef:
              name: eventhub-secret
              key: connection-string
        - name: EVENTHUB_NAME
          value: inference-requests
        - name: CONSUMER_GROUP
          value: inference-consumer
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
```

Deploy:
```bash
kubectl apply -f inference-service-deployment.yaml

# Verify deployment
kubectl get pods -n mlflow -l app=inference-service
kubectl logs -f deployment/inference-service -n mlflow
```

### 6. Create Request Producer in Databricks
Create Databricks notebook:

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

### 7. Consume Results in Databricks
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

### 8. Implement Batch Inference with Spark Streaming
Create Spark Structured Streaming job:

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

### 10. Implement Model Versioning in Pipeline
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

### 11. Add Feature Preprocessing Pipeline
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

### 12. Implement Caching Layer with Redis
Deploy Redis:

```bash
helm install redis bitnami/redis \
  --namespace mlflow \
  --set auth.enabled=false

export REDIS_HOST=$(kubectl get svc redis-master -n mlflow -o jsonpath='{.spec.clusterIP}')
```

Add caching to inference service:

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

### 13. Add Request Queue with Priority
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

### 14. Implement Circuit Breaker Pattern
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

### 15. Create Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-service-hpa
  namespace: mlflow
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
  - type: Pods
    pods:
      metric:
        name: kafka_consumer_lag
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
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
```

### 16. Monitor with Grafana Dashboard
Create Prometheus ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: inference-service-metrics
  namespace: mlflow
spec:
  selector:
    matchLabels:
      app: inference-service
  endpoints:
  - port: metrics
    interval: 30s
```

Import Grafana dashboard for inference metrics.

### 17. Load Testing
Create load test script:

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

### 18. Cleanup
```bash
# Delete deployments
kubectl delete deployment inference-service -n mlflow
kubectl delete hpa inference-service-hpa -n mlflow

# Delete Event Hub
az eventhubs eventhub delete \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $EVENTHUB_NAMESPACE \
  --name $EVENTHUB_NAME

az eventhubs namespace delete \
  --resource-group $RESOURCE_GROUP \
  --name $EVENTHUB_NAMESPACE

# Delete Kafka
helm uninstall kafka -n kafka
kubectl delete namespace kafka

# Delete Redis
helm uninstall redis -n mlflow
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



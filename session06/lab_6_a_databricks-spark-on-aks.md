# Lab 6a: Databricks Spark on AKS

## Objective
Run Apache Spark workloads from Databricks on Azure Kubernetes Service.

## Prerequisites
- AKS cluster running
- Azure Databricks workspace
- `kubectl` configured
- Azure CLI installed
- Contributor access to AKS and Databricks

## Steps

### 1. Create Azure Databricks Workspace
```bash
# Set variables
RESOURCE_GROUP="aks-databricks-rg"
LOCATION="eastus"
DATABRICKS_WORKSPACE="aks-databricks-workspace"
AKS_CLUSTER="aks-spark-cluster"

# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Create Databricks workspace
az databricks workspace create \
  --resource-group $RESOURCE_GROUP \
  --name $DATABRICKS_WORKSPACE \
  --location $LOCATION \
  --sku premium

# Get workspace URL
DATABRICKS_URL=$(az databricks workspace show \
  --resource-group $RESOURCE_GROUP \
  --name $DATABRICKS_WORKSPACE \
  --query workspaceUrl -o tsv)

echo "Databricks URL: https://$DATABRICKS_URL"
```

### 2. Create or Use Existing AKS Cluster
```bash
# Create AKS cluster optimized for Spark workloads
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER \
  --location $LOCATION \
  --node-count 3 \
  --node-vm-size Standard_D8s_v3 \
  --enable-managed-identity \
  --network-plugin azure \
  --generate-ssh-keys

# Get credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER \
  --overwrite-existing
```

### 3. Configure AKS for Spark Workloads
Create namespace and service account:

```bash
# Create namespace for Spark jobs
kubectl create namespace spark-jobs

# Create service account
kubectl create serviceaccount spark -n spark-jobs
```

Create RBAC role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: spark-role
  namespace: spark-jobs
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spark-role-binding
  namespace: spark-jobs
subjects:
- kind: ServiceAccount
  name: spark
  namespace: spark-jobs
roleRef:
  kind: Role
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:
```bash
kubectl apply -f spark-rbac.yaml
```

### 4. Get AKS Cluster Information
```bash
# Get API server URL
AKS_API_SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

# Get cluster CA certificate
kubectl get secret \
  $(kubectl get sa spark -n spark-jobs -o jsonpath='{.secrets[0].name}') \
  -n spark-jobs \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt

# Get service account token
SA_TOKEN=$(kubectl get secret \
  $(kubectl get sa spark -n spark-jobs -o jsonpath='{.secrets[0].name}') \
  -n spark-jobs \
  -o jsonpath='{.data.token}' | base64 -d)

echo "API Server: $AKS_API_SERVER"
echo "Token saved to variable SA_TOKEN"
```

### 5. Create Databricks Cluster with AKS Integration
In Databricks UI:

1. Go to **Compute** → **Create Cluster**
2. Configure cluster:
   - **Cluster name**: `spark-on-aks`
   - **Cluster mode**: Standard
   - **Databricks Runtime**: 13.3 LTS (or latest)
   - **Worker type**: Standard_D8s_v3
   - **Min workers**: 2
   - **Max workers**: 5
   - **Enable autoscaling**: Yes

3. Add Spark configuration:
```properties
spark.kubernetes.container.image.pullPolicy Always
spark.kubernetes.namespace spark-jobs
spark.kubernetes.authenticate.driver.serviceAccountName spark
```

### 6. Configure Databricks to Access AKS
Create Databricks secret scope for AKS credentials:

```bash
# Install Databricks CLI
pip install databricks-cli

# Configure Databricks CLI
databricks configure --token

# Create secret scope
databricks secrets create-scope --scope aks-secrets

# Add AKS credentials
databricks secrets put-secret --scope aks-secrets \
  --key aks-api-server \
  --string-value "$AKS_API_SERVER"

databricks secrets put-secret --scope aks-secrets \
  --key aks-token \
  --string-value "$SA_TOKEN"
```

### 7. Create Spark Application
Create notebook in Databricks:

```python
# Sample PySpark application
from pyspark.sql import SparkSession

# Create Spark session configured for AKS
spark = SparkSession.builder \
    .appName("SparkOnAKS") \
    .config("spark.kubernetes.container.image", "databricksruntime/standard:13.3-LTS") \
    .config("spark.kubernetes.namespace", "spark-jobs") \
    .config("spark.kubernetes.authenticate.driver.serviceAccountName", "spark") \
    .config("spark.executor.instances", "3") \
    .config("spark.executor.memory", "4g") \
    .config("spark.executor.cores", "2") \
    .getOrCreate()

# Sample DataFrame operations
data = [("Alice", 34), ("Bob", 45), ("Charlie", 28)]
df = spark.createDataFrame(data, ["name", "age"])

# Perform transformations
result = df.filter(df.age > 30).select("name", "age")
result.show()

# Stop Spark session
spark.stop()
```

### 8. Submit Spark Job to AKS via Databricks
Create Python script `spark_job.py`:

```python
from pyspark.sql import SparkSession
import sys

def main():
    # Initialize Spark session
    spark = SparkSession.builder \
        .appName("AKS-Spark-Job") \
        .getOrCreate()
    
    # Sample data processing
    data = range(1, 1000000)
    rdd = spark.sparkContext.parallelize(data)
    
    # Perform operations
    result = rdd.map(lambda x: x * 2).filter(lambda x: x % 3 == 0).count()
    
    print(f"Result: {result}")
    
    spark.stop()

if __name__ == "__main__":
    main()
```

### 9. Use spark-submit from Databricks
Create notebook cell:

```python
import subprocess
import os

# Set environment variables
os.environ['KUBERNETES_MASTER'] = dbutils.secrets.get(scope="aks-secrets", key="aks-api-server")
os.environ['KUBERNETES_AUTH_TOKEN'] = dbutils.secrets.get(scope="aks-secrets", key="aks-token")

# Submit Spark job to AKS
spark_submit_cmd = """
spark-submit \
  --master k8s://{} \
  --deploy-mode cluster \
  --name spark-aks-job \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
  --conf spark.kubernetes.namespace=spark-jobs \
  --conf spark.executor.instances=3 \
  --conf spark.executor.memory=4g \
  --conf spark.executor.cores=2 \
  --conf spark.kubernetes.container.image=apache/spark:3.4.1 \
  local:///opt/spark/examples/src/main/python/pi.py 100
""".format(os.environ['KUBERNETES_MASTER'])

result = subprocess.run(spark_submit_cmd, shell=True, capture_output=True, text=True)
print(result.stdout)
print(result.stderr)
```

### 10. Monitor Spark Jobs on AKS
```bash
# Watch Spark driver pods
kubectl get pods -n spark-jobs -w

# View logs from Spark driver
DRIVER_POD=$(kubectl get pods -n spark-jobs -l spark-role=driver --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
kubectl logs -f $DRIVER_POD -n spark-jobs

# View executor pods
kubectl get pods -n spark-jobs -l spark-role=executor

# Check Spark UI
kubectl port-forward -n spark-jobs $DRIVER_POD 4040:4040
# Access at http://localhost:4040
```

### 11. Create Databricks Job for AKS Spark
In Databricks UI:

1. Go to **Workflows** → **Create Job**
2. Configure:
   - **Task name**: `spark-on-aks-task`
   - **Type**: Python script or Notebook
   - **Cluster**: Use existing `spark-on-aks` cluster
   - **Parameters**: Add as needed

3. Advanced settings:
```json
{
  "spark_conf": {
    "spark.kubernetes.namespace": "spark-jobs",
    "spark.kubernetes.authenticate.driver.serviceAccountName": "spark",
    "spark.executor.instances": "3"
  }
}
```

### 12. Configure Resource Quotas for Spark
Create resource quota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: spark-quota
  namespace: spark-jobs
spec:
  hard:
    requests.cpu: "32"
    requests.memory: 64Gi
    limits.cpu: "64"
    limits.memory: 128Gi
    pods: "50"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: spark-limits
  namespace: spark-jobs
spec:
  limits:
  - max:
      cpu: "8"
      memory: 16Gi
    min:
      cpu: "1"
      memory: 1Gi
    type: Container
```

Apply:
```bash
kubectl apply -f spark-quotas.yaml
```

### 13. Enable Dynamic Allocation
Configure Spark for dynamic executor allocation:

```python
spark = SparkSession.builder \
    .appName("DynamicAllocationApp") \
    .config("spark.dynamicAllocation.enabled", "true") \
    .config("spark.dynamicAllocation.shuffleTracking.enabled", "true") \
    .config("spark.dynamicAllocation.minExecutors", "1") \
    .config("spark.dynamicAllocation.maxExecutors", "10") \
    .config("spark.dynamicAllocation.initialExecutors", "2") \
    .config("spark.kubernetes.namespace", "spark-jobs") \
    .getOrCreate()
```

### 14. Configure Persistent Storage for Spark
Create PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spark-storage
  namespace: spark-jobs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile
  resources:
    requests:
      storage: 100Gi
```

Mount in Spark configuration:

```python
spark_conf = {
    "spark.kubernetes.driver.volumes.persistentVolumeClaim.spark-storage.options.claimName": "spark-storage",
    "spark.kubernetes.driver.volumes.persistentVolumeClaim.spark-storage.mount.path": "/data",
    "spark.kubernetes.executor.volumes.persistentVolumeClaim.spark-storage.options.claimName": "spark-storage",
    "spark.kubernetes.executor.volumes.persistentVolumeClaim.spark-storage.mount.path": "/data"
}
```

### 15. Connect to Azure Data Lake from Spark
Configure Spark to access ADLS:

```python
from pyspark.sql import SparkSession

# Set up ADLS credentials
storage_account = "<storage-account-name>"
container = "<container-name>"
storage_key = "<storage-key>"

spark = SparkSession.builder \
    .appName("SparkADLS") \
    .config(f"fs.azure.account.key.{storage_account}.dfs.core.windows.net", storage_key) \
    .getOrCreate()

# Read from ADLS
df = spark.read.parquet(f"abfss://{container}@{storage_account}.dfs.core.windows.net/data/")
df.show()

# Write to ADLS
df.write.mode("overwrite").parquet(f"abfss://{container}@{storage_account}.dfs.core.windows.net/output/")
```

### 16. Use Azure Managed Identity for Authentication
Configure AKS pod identity:

```bash
# Enable pod identity on AKS
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER \
  --enable-pod-identity

# Create managed identity
IDENTITY_NAME="spark-identity"
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME

# Get identity details
IDENTITY_CLIENT_ID=$(az identity show --resource-group $RESOURCE_GROUP --name $IDENTITY_NAME --query clientId -o tsv)
IDENTITY_RESOURCE_ID=$(az identity show --resource-group $RESOURCE_GROUP --name $IDENTITY_NAME --query id -o tsv)

# Create pod identity binding
az aks pod-identity add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_CLUSTER \
  --namespace spark-jobs \
  --name spark-pod-identity \
  --identity-resource-id $IDENTITY_RESOURCE_ID
```

Use in Spark:

```python
spark = SparkSession.builder \
    .config("spark.kubernetes.driver.label.aadpodidbinding", "spark-pod-identity") \
    .config("spark.kubernetes.executor.label.aadpodidbinding", "spark-pod-identity") \
    .getOrCreate()
```

### 17. Monitor Spark Performance
Create monitoring dashboard in Databricks:

```python
# Get Spark metrics
spark.sparkContext.statusTracker().getExecutorInfos()

# Monitor job progress
from pyspark import SparkContext
sc = SparkContext.getOrCreate()
status = sc.statusTracker()

for job_id in status.getActiveJobIds():
    job_info = status.getJobInfo(job_id)
    print(f"Job {job_id}: {job_info.status()}")
```

### 18. Optimize Spark on AKS
Configure optimal settings:

```python
spark_config = {
    # Memory management
    "spark.executor.memory": "8g",
    "spark.executor.memoryOverhead": "2g",
    "spark.driver.memory": "4g",
    "spark.driver.memoryOverhead": "1g",
    
    # CPU allocation
    "spark.executor.cores": "4",
    "spark.executor.instances": "5",
    
    # Kubernetes specific
    "spark.kubernetes.executor.request.cores": "4",
    "spark.kubernetes.executor.limit.cores": "4",
    
    # Performance tuning
    "spark.sql.shuffle.partitions": "200",
    "spark.default.parallelism": "100",
    "spark.sql.adaptive.enabled": "true",
    "spark.sql.adaptive.coalescePartitions.enabled": "true",
    
    # Network optimization
    "spark.network.timeout": "600s",
    "spark.executor.heartbeatInterval": "60s"
}
```

### 19. Use Spot Instances for Executors
Create node pool with spot instances:

```bash
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_CLUSTER \
  --name sparkspot \
  --node-count 3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --node-vm-size Standard_D8s_v3 \
  --labels workload=spark tier=spot \
  --node-taints spot=true:NoSchedule
```

Configure Spark to use spot nodes:

```python
spark_conf = {
    "spark.kubernetes.executor.node.selector.workload": "spark",
    "spark.kubernetes.executor.node.selector.tier": "spot",
    "spark.kubernetes.executor.tolerations": "spot=true:NoSchedule"
}
```

### 20. Cleanup
```bash
# Delete Spark jobs
kubectl delete all --all -n spark-jobs

# Delete namespace
kubectl delete namespace spark-jobs

# Delete Databricks workspace
az databricks workspace delete \
  --resource-group $RESOURCE_GROUP \
  --name $DATABRICKS_WORKSPACE \
  --yes

# Delete AKS cluster (if created for this lab)
az aks delete \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER \
  --yes --no-wait

# Delete resource group
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

## Expected Results
- Databricks workspace connected to AKS cluster
- Spark jobs running on AKS from Databricks
- Driver and executor pods scheduled in Kubernetes
- Access to Azure storage from Spark applications
- Dynamic executor allocation working
- Monitoring and logging integrated
- Cost optimization with spot instances

## Key Takeaways
- **Databricks** can orchestrate Spark jobs on AKS
- **Spark on Kubernetes** provides containerized execution
- **Service accounts** control Spark pod permissions
- **Dynamic allocation** optimizes resource usage
- **Persistent volumes** enable data sharing
- **Managed identity** simplifies authentication
- **Spot instances** reduce compute costs
- **Resource quotas** prevent resource exhaustion
- Integration with Azure Data Lake Storage
- Monitoring through Databricks and Kubernetes

## Spark on Kubernetes Architecture

| Component | Purpose |
|-----------|---------|
| Driver Pod | Spark application master |
| Executor Pods | Task execution workers |
| Service Account | RBAC authentication |
| ConfigMaps | Spark configuration |
| Persistent Volumes | Data persistence |

## Performance Tuning

| Setting | Purpose |
|---------|---------|
| executor.memory | Executor heap size |
| executor.cores | CPU cores per executor |
| executor.instances | Number of executors |
| shuffle.partitions | Shuffle parallelism |
| adaptive.enabled | Adaptive query execution |

## Troubleshooting
- **Driver pod fails**: Check service account permissions
- **Executors not starting**: Verify resource quotas
- **Storage access fails**: Check managed identity roles
- **OOM errors**: Increase executor memory overhead
- **Slow performance**: Tune shuffle partitions and parallelism

---

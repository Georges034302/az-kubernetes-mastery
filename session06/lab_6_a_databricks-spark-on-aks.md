# Lab 6a: Databricks Spark on AKS
<img width="1438" height="915" alt="ZIMAGE" src="https://github.com/user-attachments/assets/9d2c0c27-ec2d-49f2-8d4d-261a3f221324" />

## Objective
Deploy Azure Databricks and configure it to run Apache Spark workloads on Azure Kubernetes Service (AKS). Master the integration between Databricks and AKS for scalable, cost-effective Spark job execution with dynamic resource allocation and Azure storage integration.

---

## Lab Parameters

Set these variables at the start:

```bash
# Azure Resources
RESOURCE_GROUP="rg-aks-databricks-spark"
LOCATION="australiaeast"
CLUSTER_NAME="aks-spark-cluster"
NODE_COUNT=3
NODE_SIZE="Standard_D8s_v3"
K8S_VERSION="1.28"

# Databricks Configuration
DATABRICKS_WORKSPACE="databricks-spark-workspace"
DATABRICKS_SKU="premium"

# Spark Configuration
SPARK_NAMESPACE="spark-jobs"
SPARK_SA="spark"
SPARK_EXECUTOR_INSTANCES="3"
SPARK_EXECUTOR_MEMORY="4g"
SPARK_EXECUTOR_CORES="2"

# Managed Identity
IDENTITY_NAME="spark-identity"

# Storage (optional - for ADLS integration)
STORAGE_ACCOUNT="sparkstorageaks"
CONTAINER_NAME="spark-data"
```

---

## Step 1: Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \      # `Resource group name`
  --location $LOCATION          # `Azure region (Sydney, Australia)`
```

---

## Step 2: Create Azure Databricks Workspace

```bash
az databricks workspace create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --name $DATABRICKS_WORKSPACE \           # `Workspace name`
  --location $LOCATION \                   # `Region`
  --sku $DATABRICKS_SKU                    # `SKU (premium for advanced features)`

az databricks workspace wait \
  --resource-group $RESOURCE_GROUP \
  --name $DATABRICKS_WORKSPACE \
  --created                                # `Wait for workspace creation`

DATABRICKS_URL=$(az databricks workspace show \
  --resource-group $RESOURCE_GROUP \
  --name $DATABRICKS_WORKSPACE \
  --query workspaceUrl \
  -o tsv)                                  # `Get workspace URL`

echo "Databricks workspace created: $DATABRICKS_WORKSPACE"
echo "Databricks URL: https://$DATABRICKS_URL"
```

---

## Step 3: Create AKS Cluster for Spark Workloads

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \       # `Resource group`
  --name $CLUSTER_NAME \                   # `Cluster name`
  --location $LOCATION \                   # `Region`
  --node-count $NODE_COUNT \               # `Number of nodes`
  --node-vm-size $NODE_SIZE \              # `VM size (8 vCPU, 32GB for Spark)`
  --kubernetes-version $K8S_VERSION \      # `Kubernetes version`
  --enable-managed-identity \              # `Use managed identity`
  --generate-ssh-keys \                    # `Generate SSH keys`
  --network-plugin azure \                 # `Azure CNI networking`
  --no-wait                                # `Don't wait for completion`

az aks wait \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --created                                # `Wait for cluster creation`

az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing                     # `Configure kubectl`

echo "AKS cluster created: $CLUSTER_NAME"
```

---

## Step 4: Configure AKS for Spark Workloads

Create namespace and service account:
```bash
kubectl create namespace $SPARK_NAMESPACE  # `Create namespace for Spark jobs`

kubectl create serviceaccount $SPARK_SA \
  --namespace $SPARK_NAMESPACE             # `Create service account for Spark`

echo "Namespace and service account created"
```

Create RBAC role and binding:
```bash
kubectl apply --namespace $SPARK_NAMESPACE -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: spark-role
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
subjects:
- kind: ServiceAccount
  name: $SPARK_SA
  namespace: $SPARK_NAMESPACE
roleRef:
  kind: Role
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
EOF

echo "RBAC role and binding created"
```

---

## Step 5: Get AKS Cluster Information

Get API server URL:
```bash
AKS_API_SERVER=$(kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}')  # `Get API server endpoint`

echo "AKS API Server: $AKS_API_SERVER"
```

Create service account token (Kubernetes 1.24+):
```bash
kubectl apply --namespace $SPARK_NAMESPACE -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: $SPARK_SA-token
  annotations:
    kubernetes.io/service-account.name: $SPARK_SA
type: kubernetes.io/service-account-token
EOF

kubectl wait \
  --for=jsonpath='{.data.token}' \
  secret/$SPARK_SA-token \
  --namespace $SPARK_NAMESPACE \
  --timeout=60s                            # `Wait for token generation`

SA_TOKEN=$(kubectl get secret $SPARK_SA-token \
  --namespace $SPARK_NAMESPACE \
  -o jsonpath='{.data.token}' | base64 -d)  # `Get service account token`

echo "Service account token captured"
echo "Token length: ${#SA_TOKEN} characters"
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

---

## Step 6: Configure Databricks to Access AKS

Install Databricks CLI:
```bash
pip install databricks-cli  # `Install Databricks command-line interface`

echo "Databricks CLI installed"
```

Configure Databricks CLI:
```bash
# You'll be prompted for:
# - Databricks Host: https://$DATABRICKS_URL
# - Token: Generate from Databricks UI -> User Settings -> Access Tokens

databricks configure --token  # `Configure Databricks CLI with token`
```

Create secret scope and store AKS credentials:
```bash
databricks secrets create-scope \
  --scope aks-secrets  # `Create secret scope for AKS credentials`

databricks secrets put-secret \
  --scope aks-secrets \
  --key aks-api-server \
  --string-value "$AKS_API_SERVER"  # `Store API server URL`

databricks secrets put-secret \
  --scope aks-secrets \
  --key aks-token \
  --string-value "$SA_TOKEN"  # `Store service account token`

echo "AKS credentials stored in Databricks secrets"
```

---

## Step 7: Create Databricks Cluster

**In Databricks UI:**

1. Navigate to **Compute** → **Create Cluster**
2. Configure cluster:
   - **Cluster name**: `spark-on-aks`
   - **Cluster mode**: Standard
   - **Databricks Runtime**: 13.3 LTS or latest
   - **Worker type**: `Standard_D8s_v3`
   - **Min workers**: 2
   - **Max workers**: 5
   - **Enable autoscaling**: Yes

3. Add Spark configuration (Advanced Options → Spark):
```properties
spark.kubernetes.container.image.pullPolicy Always
spark.kubernetes.namespace spark-jobs
spark.kubernetes.authenticate.driver.serviceAccountName spark
```

4. Click **Create Cluster**

---

## Step 8: Create Sample Spark Application

**Create notebook in Databricks with this PySpark code:

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

---

## Step 10: Monitor Spark Jobs on AKS

Watch Spark pods:
```bash
kubectl get pods \
  --namespace $SPARK_NAMESPACE \
  --watch  # `Watch Spark driver and executor pods`
```

View Spark driver logs:
```bash
DRIVER_POD=$(kubectl get pods \
  --namespace $SPARK_NAMESPACE \
  -l spark-role=driver \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1].metadata.name}')  # `Get latest driver pod`

kubectl logs \
  --namespace $SPARK_NAMESPACE \
  --follow $DRIVER_POD  # `Follow driver logs`

echo "Driver pod: $DRIVER_POD"
```

View executor pods:
```bash
kubectl get pods \
  --namespace $SPARK_NAMESPACE \
  -l spark-role=executor  # `List executor pods`
```

Access Spark UI:
```bash
kubectl port-forward \
  --namespace $SPARK_NAMESPACE \
  --address 0.0.0.0 \
  $DRIVER_POD 4040:4040 &  # `Port-forward to Spark UI`

echo "Spark UI: http://localhost:4040"
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

---

## Step 9: Configure Resource Quotas for Spark

Create resource quota and limits:
```bash
kubectl apply --namespace $SPARK_NAMESPACE -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: spark-quota
spec:
  hard:
    requests.cpu: "32"         # Max 32 vCPUs requested
    requests.memory: 64Gi      # Max 64GB RAM requested
    limits.cpu: "64"           # Max 64 vCPUs limit
    limits.memory: 128Gi       # Max 128GB RAM limit
    pods: "50"                 # Max 50 pods
---
apiVersion: v1
kind: LimitRange
metadata:
  name: spark-limits
spec:
  limits:
  - max:
      cpu: "8"                 # Max 8 vCPUs per container
      memory: 16Gi             # Max 16GB per container
    min:
      cpu: "1"                 # Min 1 vCPU per container
      memory: 1Gi              # Min 1GB per container
    type: Container
EOF

echo "Resource quotas and limits configured"
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

---

## Step 11: Configure Persistent Storage for Spark

Create PersistentVolumeClaim:
```bash
kubectl apply --namespace $SPARK_NAMESPACE -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spark-storage
spec:
  accessModes:
    - ReadWriteMany       # Multiple pods can mount
  storageClassName: azurefile  # Azure Files for RWX
  resources:
    requests:
      storage: 100Gi       # 100GB storage
EOF

kubectl wait \
  --for=jsonpath='{.status.phase}'=Bound \
  pvc/spark-storage \
  --namespace $SPARK_NAMESPACE \
  --timeout=300s  # `Wait for PVC to be bound`

echo "Persistent storage configured"
```

**Mount in Spark configuration (use in Databricks notebook):

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

---

## Step 12: Configure Azure Managed Identity

Enable pod identity on AKS:
```bash
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --enable-pod-identity  # `Enable workload identity`

echo "Pod identity enabled on AKS"
```

Create managed identity:
```bash
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME  # `Create managed identity for Spark`

IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query clientId \
  -o tsv)  # `Get client ID`

IDENTITY_RESOURCE_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query id \
  -o tsv)  # `Get resource ID`

echo "Managed identity created: $IDENTITY_NAME"
echo "Client ID: $IDENTITY_CLIENT_ID"
```

Create pod identity binding:
```bash
az aks pod-identity add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --namespace $SPARK_NAMESPACE \
  --name spark-pod-identity \
  --identity-resource-id $IDENTITY_RESOURCE_ID  # `Bind identity to namespace`

echo "Pod identity binding created"
```

**Use in Spark (Databricks notebook):

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

---

## Step 13: Add Spot Instance Node Pool for Cost Optimization

Create spot instance node pool:
```bash
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name sparkspot \
  --node-count 3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --node-vm-size $NODE_SIZE \
  --labels workload=spark tier=spot \
  --node-taints spot=true:NoSchedule  # `Taint to prevent non-Spark workloads`

echo "Spot node pool 'sparkspot' created"
```

**Configure Spark to use spot nodes (Databricks notebook):

```python
spark_conf = {
    "spark.kubernetes.executor.node.selector.workload": "spark",
    "spark.kubernetes.executor.node.selector.tier": "spot",
    "spark.kubernetes.executor.tolerations": "spot=true:NoSchedule"
}
```

---

## Cleanup

**Option 1** - Remove Spark resources only:
```bash
kubectl delete all --all \
  --namespace $SPARK_NAMESPACE  # `Delete all Spark jobs and pods`

kubectl delete namespace $SPARK_NAMESPACE  # `Delete Spark namespace`

echo "Spark resources removed"
```

**Option 2** - Delete entire resource group:
```bash
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait  # `Delete RG (includes Databricks, AKS, storage)`

echo "Resource group deletion initiated"
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

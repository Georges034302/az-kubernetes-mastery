# Lab 6b: Databricks Delta Lake on Kubernetes

## Objective
Use Delta Lake from Databricks with Kubernetes-based Spark for ACID transactions and data versioning.

## Prerequisites
- AKS cluster running
- Azure Databricks workspace
- Azure Data Lake Storage Gen2
- Lab 6a completed (Spark on AKS setup)
- `kubectl` configured

## Steps

### 1. Set Up Azure Data Lake Storage Gen2
```bash
# Set variables
RESOURCE_GROUP="aks-databricks-rg"
STORAGE_ACCOUNT="aksdeltastorage$RANDOM"
CONTAINER="delta-lake"
LOCATION="eastus"

# Create storage account with hierarchical namespace
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true

# Create container
az storage container create \
  --name $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --resource-group $RESOURCE_GROUP \
  --account-name $STORAGE_ACCOUNT \
  --query '[0].value' -o tsv)

echo "Storage Account: $STORAGE_ACCOUNT"
echo "Container: $CONTAINER"
```

### 2. Configure Kubernetes Secret for Storage Access
```bash
# Create secret with storage credentials
kubectl create secret generic delta-storage-secret \
  --from-literal=account-name=$STORAGE_ACCOUNT \
  --from-literal=account-key=$STORAGE_KEY \
  -n spark-jobs

# Verify secret
kubectl get secret delta-storage-secret -n spark-jobs
```

### 3. Create Databricks Cluster with Delta Lake
In Databricks UI, create cluster with:

**Spark Config:**
```properties
spark.jars.packages io.delta:delta-core_2.12:2.4.0
spark.sql.extensions io.delta.sql.DeltaSparkSessionExtension
spark.sql.catalog.spark_catalog org.apache.spark.sql.delta.catalog.DeltaCatalog
spark.databricks.delta.retentionDurationCheck.enabled false
```

**Environment Variables:**
```properties
DELTA_STORAGE_ACCOUNT=$STORAGE_ACCOUNT
DELTA_CONTAINER=$CONTAINER
```

### 4. Initialize Delta Lake in Databricks
Create notebook with Delta Lake setup:

```python
from pyspark.sql import SparkSession
from delta import *

# Initialize Spark with Delta Lake
spark = SparkSession.builder \
    .appName("DeltaLakeOnAKS") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

# Configure ADLS access
storage_account = "<storage-account-name>"
storage_key = "<storage-key>"
container = "delta-lake"

spark.conf.set(
    f"fs.azure.account.key.{storage_account}.dfs.core.windows.net",
    storage_key
)

# Define Delta Lake path
delta_path = f"abfss://{container}@{storage_account}.dfs.core.windows.net/delta/tables"

print(f"Delta Lake initialized at: {delta_path}")
```

### 5. Create Delta Table
```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType
from datetime import datetime

# Create sample data
data = [
    ("customer_001", "Alice", 28, datetime.now()),
    ("customer_002", "Bob", 35, datetime.now()),
    ("customer_003", "Charlie", 42, datetime.now()),
]

schema = StructType([
    StructField("customer_id", StringType(), False),
    StructField("name", StringType(), True),
    StructField("age", IntegerType(), True),
    StructField("created_at", TimestampType(), True)
])

df = spark.createDataFrame(data, schema)

# Write as Delta table
delta_table_path = f"{delta_path}/customers"
df.write.format("delta").mode("overwrite").save(delta_table_path)

print(f"Delta table created at: {delta_table_path}")
```

### 6. Perform ACID Operations
```python
from delta.tables import DeltaTable

# Load Delta table
delta_table = DeltaTable.forPath(spark, delta_table_path)

# INSERT - Add new records
new_data = [
    ("customer_004", "Diana", 31, datetime.now()),
    ("customer_005", "Eve", 29, datetime.now())
]
new_df = spark.createDataFrame(new_data, schema)
new_df.write.format("delta").mode("append").save(delta_table_path)

# UPDATE - Modify existing records
delta_table.update(
    condition = "customer_id = 'customer_001'",
    set = {"age": "29"}
)

# DELETE - Remove records
delta_table.delete("age < 30")

# MERGE - Upsert operation
updates = [
    ("customer_002", "Bob Smith", 36, datetime.now()),
    ("customer_006", "Frank", 45, datetime.now())
]
updates_df = spark.createDataFrame(updates, schema)

delta_table.alias("target").merge(
    updates_df.alias("source"),
    "target.customer_id = source.customer_id"
).whenMatchedUpdate(
    set = {
        "name": "source.name",
        "age": "source.age",
        "created_at": "source.created_at"
    }
).whenNotMatchedInsert(
    values = {
        "customer_id": "source.customer_id",
        "name": "source.name",
        "age": "source.age",
        "created_at": "source.created_at"
    }
).execute()

# Read current data
df_current = spark.read.format("delta").load(delta_table_path)
df_current.show()
```

### 7. Time Travel Queries
```python
# View table history
delta_table.history().show()

# Read previous version by version number
df_v0 = spark.read.format("delta").option("versionAsOf", 0).load(delta_table_path)
df_v0.show()

# Read data as of specific timestamp
from datetime import datetime, timedelta
timestamp = (datetime.now() - timedelta(hours=1)).strftime("%Y-%m-%d %H:%M:%S")
df_historical = spark.read.format("delta") \
    .option("timestampAsOf", timestamp) \
    .load(delta_table_path)
df_historical.show()

# Compare versions
print("Version 0:")
df_v0.count()
print("Current version:")
df_current.count()
```

### 8. Schema Evolution
```python
from pyspark.sql.types import BooleanType

# Add new column with schema evolution
new_schema_data = [
    ("customer_007", "Grace", 33, datetime.now(), True)
]

new_schema = StructType([
    StructField("customer_id", StringType(), False),
    StructField("name", StringType(), True),
    StructField("age", IntegerType(), True),
    StructField("created_at", TimestampType(), True),
    StructField("is_premium", BooleanType(), True)
])

df_new_schema = spark.createDataFrame(new_schema_data, new_schema)

# Enable schema evolution
df_new_schema.write.format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .save(delta_table_path)

# Verify schema change
spark.read.format("delta").load(delta_table_path).printSchema()
```

### 9. Optimize Delta Tables
```python
# Optimize - Compact small files
delta_table.optimize().executeCompaction()

# Z-ordering for query performance
delta_table.optimize().executeZOrderBy("customer_id")

# Vacuum - Remove old files
delta_table.vacuum(168)  # Retain 7 days of history

# Check table details
delta_table.detail().show()
```

### 10. Create Delta Table with Partitioning
```python
# Create partitioned Delta table
orders_data = [
    ("order_001", "customer_001", 100.50, "2024-01-15", "electronics"),
    ("order_002", "customer_002", 250.00, "2024-01-16", "clothing"),
    ("order_003", "customer_001", 75.25, "2024-02-10", "electronics"),
    ("order_004", "customer_003", 300.00, "2024-02-12", "furniture"),
]

orders_schema = StructType([
    StructField("order_id", StringType(), False),
    StructField("customer_id", StringType(), True),
    StructField("amount", StringType(), True),
    StructField("order_date", StringType(), True),
    StructField("category", StringType(), True)
])

orders_df = spark.createDataFrame(orders_data, orders_schema)

# Write with partitioning
orders_table_path = f"{delta_path}/orders"
orders_df.write.format("delta") \
    .mode("overwrite") \
    .partitionBy("category") \
    .save(orders_table_path)

# Verify partitions
spark.read.format("delta").load(orders_table_path).show()
```

### 11. Streaming with Delta Lake
```python
from pyspark.sql.functions import col, current_timestamp

# Create streaming source (simulated)
streaming_data_path = f"{delta_path}/streaming_source"

# Write sample streaming data
for i in range(5):
    batch_data = [
        (f"event_{i}_{j}", f"user_{j}", datetime.now())
        for j in range(10)
    ]
    batch_df = spark.createDataFrame(
        batch_data,
        ["event_id", "user_id", "timestamp"]
    )
    batch_df.write.format("delta").mode("append").save(streaming_data_path)

# Read stream from Delta table
stream_df = spark.readStream.format("delta").load(streaming_data_path)

# Write stream to Delta table
events_table_path = f"{delta_path}/events"
query = stream_df.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", f"{delta_path}/checkpoints/events") \
    .start(events_table_path)

# Let it run for a few seconds
import time
time.sleep(10)
query.stop()

# Verify streaming data
spark.read.format("delta").load(events_table_path).count()
```

### 12. Change Data Feed
```python
# Enable change data feed on table
spark.sql(f"""
    ALTER TABLE delta.`{delta_table_path}`
    SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
""")

# Perform some changes
delta_table.update(
    condition = "age > 35",
    set = {"age": "age + 1"}
)

# Read change data feed
changes_df = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", 0) \
    .load(delta_table_path)

changes_df.select("customer_id", "name", "_change_type", "_commit_version").show()
```

### 13. Delta Lake on Kubernetes Job
Create Kubernetes job that uses Delta Lake:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: delta-processing-job
  namespace: spark-jobs
spec:
  template:
    spec:
      serviceAccountName: spark
      containers:
      - name: spark-delta
        image: apache/spark:3.4.1
        command:
        - /opt/spark/bin/spark-submit
        - --master
        - local[*]
        - --packages
        - io.delta:delta-core_2.12:2.4.0
        - --conf
        - spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension
        - --conf
        - spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog
        - /app/delta_job.py
        env:
        - name: STORAGE_ACCOUNT
          valueFrom:
            secretKeyRef:
              name: delta-storage-secret
              key: account-name
        - name: STORAGE_KEY
          valueFrom:
            secretKeyRef:
              name: delta-storage-secret
              key: account-key
        volumeMounts:
        - name: app-code
          mountPath: /app
      volumes:
      - name: app-code
        configMap:
          name: delta-job-code
      restartPolicy: Never
  backoffLimit: 3
```

Create ConfigMap with job code:

```bash
cat > delta_job.py <<'EOF'
from pyspark.sql import SparkSession
import os

storage_account = os.environ['STORAGE_ACCOUNT']
storage_key = os.environ['STORAGE_KEY']

spark = SparkSession.builder \
    .appName("DeltaK8sJob") \
    .getOrCreate()

spark.conf.set(
    f"fs.azure.account.key.{storage_account}.dfs.core.windows.net",
    storage_key
)

# Process Delta table
delta_path = f"abfss://delta-lake@{storage_account}.dfs.core.windows.net/delta/tables/customers"
df = spark.read.format("delta").load(delta_path)
result = df.groupBy("age").count()
result.show()
EOF

kubectl create configmap delta-job-code \
  --from-file=delta_job.py \
  -n spark-jobs
```

Run the job:
```bash
kubectl apply -f delta-processing-job.yaml

# Monitor job
kubectl get jobs -n spark-jobs -w
kubectl logs -f job/delta-processing-job -n spark-jobs
```

### 14. Delta Lake Table Constraints
```python
# Add CHECK constraint
spark.sql(f"""
    ALTER TABLE delta.`{delta_table_path}`
    ADD CONSTRAINT age_check CHECK (age >= 18 AND age <= 120)
""")

# Add NOT NULL constraint
spark.sql(f"""
    ALTER TABLE delta.`{delta_table_path}`
    ALTER COLUMN customer_id SET NOT NULL
""")

# Try to insert invalid data (will fail)
try:
    invalid_data = [("customer_008", "Invalid", 10, datetime.now())]
    invalid_df = spark.createDataFrame(invalid_data, schema)
    invalid_df.write.format("delta").mode("append").save(delta_table_path)
except Exception as e:
    print(f"Constraint violation: {e}")

# View constraints
spark.sql(f"DESCRIBE DETAIL delta.`{delta_table_path}`").show(truncate=False)
```

### 15. Delta Lake with Multiple Environments
Create separate Delta paths for dev/staging/prod:

```python
environments = {
    "dev": f"{delta_path}/dev",
    "staging": f"{delta_path}/staging",
    "prod": f"{delta_path}/prod"
}

# Promote data across environments
def promote_delta_table(source_env, target_env, table_name):
    source_path = f"{environments[source_env]}/{table_name}"
    target_path = f"{environments[target_env]}/{table_name}"
    
    # Read from source
    df = spark.read.format("delta").load(source_path)
    
    # Write to target
    df.write.format("delta").mode("overwrite").save(target_path)
    
    print(f"Promoted {table_name} from {source_env} to {target_env}")

# Example: Promote from dev to staging
promote_delta_table("dev", "staging", "customers")
```

### 16. Delta Lake Metrics and Monitoring
```python
# Get detailed metrics
detail_df = spark.sql(f"DESCRIBE DETAIL delta.`{delta_table_path}`")
detail_df.select("format", "numFiles", "sizeInBytes", "properties").show(truncate=False)

# Get table statistics
spark.sql(f"DESCRIBE EXTENDED delta.`{delta_table_path}`").show(truncate=False)

# Get history with operation metrics
history_df = delta_table.history()
history_df.select("version", "timestamp", "operation", "operationMetrics").show(truncate=False)

# File-level information
files_df = spark.sql(f"SELECT * FROM delta.`{delta_table_path}`.files")
files_df.show()
```

### 17. Clone Delta Tables
```python
# Shallow clone (metadata only)
shallow_clone_path = f"{delta_path}/customers_clone_shallow"
spark.sql(f"""
    CREATE TABLE delta.`{shallow_clone_path}`
    SHALLOW CLONE delta.`{delta_table_path}`
""")

# Deep clone (full copy)
deep_clone_path = f"{delta_path}/customers_clone_deep"
spark.sql(f"""
    CREATE TABLE delta.`{deep_clone_path}`
    DEEP CLONE delta.`{delta_table_path}`
""")

# Verify clones
spark.read.format("delta").load(shallow_clone_path).count()
spark.read.format("delta").load(deep_clone_path).count()
```

### 18. Delta Lake with External Tables
```python
# Create external table in Hive metastore
spark.sql(f"""
    CREATE TABLE IF NOT EXISTS customers_external
    USING DELTA
    LOCATION '{delta_table_path}'
""")

# Query using table name
spark.sql("SELECT * FROM customers_external WHERE age > 30").show()

# Drop table (retains data)
spark.sql("DROP TABLE IF EXISTS customers_external")
```

### 19. Performance Tuning for Delta Lake
```python
# Configure Delta Lake optimizations
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
spark.conf.set("spark.databricks.delta.properties.defaults.autoOptimize.optimizeWrite", "true")
spark.conf.set("spark.databricks.delta.properties.defaults.autoOptimize.autoCompact", "true")

# Enable predicate pushdown
spark.conf.set("spark.databricks.delta.stats.collect", "true")

# Optimize data layout
delta_table.optimize().executeCompaction()

# Update statistics
spark.sql(f"ANALYZE TABLE delta.`{delta_table_path}` COMPUTE STATISTICS")
```

### 20. Cleanup
```bash
# Delete Delta tables in storage
az storage blob delete-batch \
  --account-name $STORAGE_ACCOUNT \
  --source $CONTAINER \
  --pattern "delta/*" \
  --auth-mode login

# Delete Kubernetes resources
kubectl delete configmap delta-job-code -n spark-jobs
kubectl delete secret delta-storage-secret -n spark-jobs
kubectl delete job delta-processing-job -n spark-jobs

# Delete storage account (optional)
# az storage account delete \
#   --name $STORAGE_ACCOUNT \
#   --resource-group $RESOURCE_GROUP \
#   --yes
```

## Expected Results
- Delta Lake tables created in ADLS Gen2
- ACID transactions working (INSERT, UPDATE, DELETE, MERGE)
- Time travel queries accessing historical data
- Schema evolution supported
- Table optimization and vacuuming functional
- Streaming writes to Delta tables
- Change data feed capturing modifications
- Kubernetes jobs processing Delta tables
- Table constraints enforced
- Performance optimized with Z-ordering

## Key Takeaways
- **Delta Lake** provides ACID transactions on data lakes
- **Time travel** enables accessing historical data versions
- **Schema evolution** allows flexible data model changes
- **MERGE** operation enables efficient upserts
- **Optimize** compacts small files for better performance
- **Z-ordering** improves query performance
- **Vacuum** removes old files to reduce storage costs
- **Change data feed** tracks data modifications
- **Constraints** enforce data quality
- Integration with Kubernetes for batch processing

## Delta Lake Operations

| Operation | Purpose |
|-----------|---------|
| INSERT | Add new records |
| UPDATE | Modify existing records |
| DELETE | Remove records |
| MERGE | Upsert (update + insert) |
| OPTIMIZE | Compact small files |
| VACUUM | Clean old versions |
| Z-ORDER | Optimize data layout |

## Time Travel Options

| Option | Usage |
|--------|-------|
| versionAsOf | Read specific version number |
| timestampAsOf | Read as of timestamp |
| history() | View all versions |
| RESTORE | Restore to previous version |

## Troubleshooting
- **Schema mismatch**: Enable mergeSchema option
- **Concurrent writes**: Delta Lake handles automatically
- **Storage access denied**: Check managed identity permissions
- **Slow queries**: Run OPTIMIZE and Z-ORDER
- **Storage costs high**: Run VACUUM to clean old files

---


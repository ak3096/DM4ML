# RecoMart Data Versioning & Lineage Documentation

## Overview
This document describes the data versioning strategy for the RecoMart recommendation pipeline using Delta Lake's built-in time travel capabilities and Unity Catalog lineage tracking.

## Delta Lake Time Travel

### What is Time Travel?
Delta Lake automatically versions data with every write operation and maintains a transaction log that allows you to query data as it existed at any point in time.

### Benefits
* **Audit and Compliance**: Track all changes to data over time
* **Rollback Capabilities**: Revert to previous versions if needed
* **Reproducibility**: Re-run analyses with historical data
* **Debug Data Issues**: Compare current vs historical data
* **A/B Testing**: Use different data versions for experiments

## Versioning Implementation

### 1. Automatic Versioning
Every write operation to our Delta tables creates a new version:

```python
# Initial write creates version 0
df.write.format("delta").mode("overwrite").saveAsTable("recomart.raw.user_interactions")

# Append creates version 1
df.write.format("delta").mode("append").saveAsTable("recomart.raw.user_interactions")

# Another append creates version 2
df.write.format("delta").mode("append").saveAsTable("recomart.raw.user_interactions")
```

### 2. Query by Version Number

```sql
-- Query version 5 of the table
SELECT * FROM recomart.raw.transactions VERSION AS OF 5;
```

```python
# Using PySpark
df = spark.read\
    .format("delta")\
    .option("versionAsOf", 5)\
    .table("recomart.raw.transactions")
```

### 3. Query by Timestamp

```sql
-- Query data as of specific date/time
SELECT * FROM recomart.raw.transactions 
TIMESTAMP AS OF '2026-04-15 10:30:00';
```

```python
# Using PySpark
df = spark.read\
    .format("delta")\
    .option("timestampAsOf", "2026-04-15 10:30:00")\
    .table("recomart.raw.transactions")
```

### 4. View Table History

```sql
-- Show all versions and operations
DESCRIBE HISTORY recomart.raw.transactions;
```

Output includes:
* Version number
* Timestamp
* Operation (WRITE, MERGE, DELETE, etc.)
* User who performed the operation
* Number of rows affected
* Metrics (files added, removed, etc.)

```python
# Using PySpark
history = spark.sql("DESCRIBE HISTORY recomart.raw.transactions")
display(history)
```

## Versioning Strategy by Table

### Raw Tables (Bronze Layer)
**Tables**: `user_interactions`, `transactions`, `products`

* **Versioning Frequency**: Every ingestion batch
* **Retention**: 30 days (configurable)
* **Use Cases**:
  * Debug ingestion issues
  * Reprocess data from specific dates
  * Audit data changes

```python
# Example: Query data from yesterday
from datetime import datetime, timedelta

yesterday = datetime.now() - timedelta(days=1)
df = spark.read\
    .format("delta")\
    .option("timestampAsOf", yesterday.isoformat())\
    .table("recomart.raw.user_interactions")
```

### Clean Tables (Silver Layer)
**Tables**: `user_interactions_clean`, `transactions_clean`, `products_clean`

* **Versioning Frequency**: Every data preparation run
* **Retention**: 60 days
* **Use Cases**:
  * Compare cleaning logic changes
  * Reproduce feature engineering with historical cleaned data
  * Validate data quality improvements

### Feature Tables (Gold Layer)
**Tables**: `user_features`, `item_features`, `user_item_features`

* **Versioning Frequency**: Every feature computation run
* **Retention**: 90 days
* **Use Cases**:
  * Model reproducibility (train with exact same features)
  * Feature drift detection
  * A/B testing different feature versions

```python
# Example: Train model with features from 7 days ago
feature_date = datetime.now() - timedelta(days=7)

user_features = spark.read\
    .format("delta")\
    .option("timestampAsOf", feature_date.isoformat())\
    .table("recomart.features.user_features")

item_features = spark.read\
    .format("delta")\
    .option("timestampAsOf", feature_date.isoformat())\
    .table("recomart.features.item_features")
```

## Data Retention & Vacuum

### Retention Configuration

By default, Delta Lake keeps 30 days of history. Configure per table:

```sql
-- Set retention to 60 days
ALTER TABLE recomart.raw.transactions 
SET TBLPROPERTIES (delta.logRetentionDuration = '60 days');

-- Set data file retention to 90 days
ALTER TABLE recomart.raw.transactions 
SET TBLPROPERTIES (delta.deletedFileRetentionDuration = '90 days');
```

### Vacuum Operation

Remove old data files to save storage costs:

```sql
-- Remove files older than retention period (default: 7 days)
VACUUM recomart.raw.transactions;

-- Vacuum with custom retention (30 days)
VACUUM recomart.raw.transactions RETAIN 720 HOURS;

-- Dry run to see what would be deleted
VACUUM recomart.raw.transactions DRY RUN;
```

**⚠️ Warning**: After VACUUM, you cannot time travel to versions older than the retention period!

### Recommended Vacuum Schedule

| Layer | Table Type | Vacuum Frequency | Retention |
|-------|-----------|------------------|-----------|
| Bronze | Raw tables | Weekly | 30 days |
| Silver | Clean tables | Weekly | 60 days |
| Gold | Feature tables | Bi-weekly | 90 days |

## Rollback & Restore

### Restore Table to Previous Version

```sql
-- Restore to version 10
RESTORE TABLE recomart.raw.transactions TO VERSION AS OF 10;

-- Restore to specific timestamp
RESTORE TABLE recomart.raw.transactions TO TIMESTAMP AS OF '2026-04-15 10:00:00';
```

```python
# Using PySpark
spark.sql("""
    RESTORE TABLE recomart.raw.transactions 
    TO VERSION AS OF 10
""")
```

### Common Rollback Scenarios

#### 1. Bad Data Ingestion
```python
# Check current version
current_version = spark.sql("DESCRIBE HISTORY recomart.raw.transactions").first().version

# Ingest new data
# ... ingestion code ...

# Data looks bad, rollback
spark.sql(f"""
    RESTORE TABLE recomart.raw.transactions 
    TO VERSION AS OF {current_version}
""")
```

#### 2. Failed Data Cleaning
```python
# Restore clean table to before cleaning run
spark.sql("""
    RESTORE TABLE recomart.clean.transactions_clean 
    TO TIMESTAMP AS OF '2026-04-20 08:00:00'
""")
```

## Unity Catalog Lineage

### Automatic Lineage Tracking

Unity Catalog automatically captures:
* **Table-to-Table Lineage**: Which tables were used to create others
* **Column-Level Lineage**: How columns are derived from source columns
* **Notebook/Query Lineage**: Which notebooks/queries accessed/modified tables
* **User Lineage**: Who accessed/modified data

### View Lineage in UI

1. Navigate to Catalog Explorer
2. Select a table (e.g., `recomart.features.user_features`)
3. Click **Lineage** tab
4. See upstream and downstream dependencies

### Programmatic Lineage Access

```python
# Get table lineage
lineage = spark.sql("""
    SELECT * FROM system.access.table_lineage
    WHERE target_table_full_name = 'recomart.features.user_features'
""")
display(lineage)

# Get column lineage
col_lineage = spark.sql("""
    SELECT * FROM system.access.column_lineage
    WHERE target_table_full_name = 'recomart.features.user_features'
""")
display(col_lineage)
```

## Version Tagging

### Tag Important Versions

Add meaningful tags to versions for easy reference:

```sql
-- Tag version used for model training
COMMENT ON TABLE recomart.features.user_features IS 
'Version 15 used for ALS model training on 2026-04-20';
```

### Version Documentation Strategy

Maintain a version log in a separate tracking table:

```python
# Create version tracking table
spark.sql("""
    CREATE TABLE IF NOT EXISTS recomart.metadata.version_log (
        table_name STRING,
        version BIGINT,
        timestamp TIMESTAMP,
        operation STRING,
        description STRING,
        user STRING,
        model_run_id STRING
    )
""")

# Log important versions
spark.sql("""
    INSERT INTO recomart.metadata.version_log VALUES (
        'recomart.features.user_features',
        15,
        current_timestamp(),
        'FEATURE_COMPUTATION',
        'Features for ALS model training - v1.2',
        current_user(),
        'run_abc123'
    )
""")
```

## Data Lineage Best Practices

### 1. Document Data Transformations

```python
# Add descriptive comments to transformation steps
df_clean = df_raw\
    .filter(F.col("user_id").isNotNull())  # Remove missing users\
    .dropDuplicates(["transaction_id"])     # Remove duplicates
    
# Log transformation metadata
transformation_metadata = {
    "source_table": "recomart.raw.transactions",
    "target_table": "recomart.clean.transactions_clean",
    "transformation": "data_cleaning",
    "filters_applied": ["not null user_id", "deduplicate transaction_id"],
    "timestamp": datetime.now().isoformat()
}
```

### 2. Version Control Pipeline Code

* Store notebooks in Git repositories
* Tag releases corresponding to data versions
* Link Git commit SHAs to data versions

### 3. Track Feature Computation Metadata

```python
# Save feature metadata
feature_metadata = {
    "feature_version": "1.2.0",
    "computation_date": datetime.now().isoformat(),
    "source_tables": [
        {"table": "recomart.raw.transactions", "version": 45},
        {"table": "recomart.raw.user_interactions", "version": 102}
    ],
    "feature_count": 25,
    "notebook": "04_Feature_Engineering_Store",
    "git_commit": "abc123def456"
}

# Store in Unity Catalog table comment
import json
spark.sql(f"""
    COMMENT ON TABLE recomart.features.user_features IS 
    '{json.dumps(feature_metadata)}'
""")
```

## Reproducibility Checklist

To ensure full reproducibility of model training:

- [ ] **Record table versions** used for training
- [ ] **Record feature computation timestamp**
- [ ] **Log model hyperparameters** in MLflow
- [ ] **Track notebook version** (Git commit SHA)
- [ ] **Document data filters and transformations**
- [ ] **Store training/test split random seed**
- [ ] **Save model evaluation metrics**
- [ ] **Archive model artifacts** in MLflow

### Example: Complete Reproducibility Metadata

```python
reproducibility_record = {
    "model_run_id": mlflow.active_run().info.run_id,
    "training_date": "2026-04-20T15:30:00Z",
    "data_versions": {
        "user_features": {
            "table": "recomart.features.user_features",
            "version": 15,
            "timestamp": "2026-04-20T08:00:00Z"
        },
        "item_features": {
            "table": "recomart.features.item_features",
            "version": 12,
            "timestamp": "2026-04-20T08:00:00Z"
        },
        "transactions": {
            "table": "recomart.clean.transactions_clean",
            "version": 45,
            "timestamp": "2026-04-20T06:00:00Z"
        }
    },
    "code_version": {
        "notebook": "05_Model_Training_Evaluation",
        "git_commit": "abc123def456",
        "git_branch": "main"
    },
    "random_seed": 42,
    "train_test_split": [0.8, 0.2],
    "model_hyperparameters": {
        "rank": 10,
        "maxIter": 10,
        "regParam": 0.1
    }
}

# Log to MLflow
mlflow.log_dict(reproducibility_record, "reproducibility.json")
```

## Monitoring & Alerts

### Set Up Version Monitoring

```python
# Check for unexpected version jumps (indicates data issues)
def check_version_health(table_name):
    history = spark.sql(f"DESCRIBE HISTORY {table_name}")
    latest_versions = history.limit(10)
    
    # Alert if more than 5 versions created in last hour
    recent = latest_versions.filter(
        F.col("timestamp") > F.current_timestamp() - F.expr("INTERVAL 1 HOUR")
    ).count()
    
    if recent > 5:
        print(f"⚠️  Warning: {recent} versions created in last hour for {table_name}")
        # Send alert to monitoring system
    
    return recent

# Run checks
check_version_health("recomart.raw.user_interactions")
check_version_health("recomart.raw.transactions")
```

---

## Summary

RecoMart's data versioning strategy leverages Delta Lake's time travel and Unity Catalog lineage to provide:

✅ **Complete Audit Trail**: Every data change is tracked  
✅ **Point-in-Time Recovery**: Query data as it existed at any point  
✅ **Model Reproducibility**: Re-train models with exact same data  
✅ **Data Lineage**: Understand data flow from source to features to models  
✅ **Rollback Capability**: Recover from data quality issues quickly  

**Document Version**: 1.0  
**Last Updated**: 2026-04-20  
**Maintained By**: RecoMart Data Platform Team

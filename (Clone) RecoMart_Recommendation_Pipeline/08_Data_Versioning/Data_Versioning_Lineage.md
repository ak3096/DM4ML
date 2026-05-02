# Data Versioning & Lineage - RecoMart Pipeline

## Overview
This document describes the data versioning strategy for the RecoMart recommendation pipeline using Delta Lake time travel capabilities and lineage tracking.

## Delta Lake Time Travel

Delta Lake provides built-in versioning through transaction logs, enabling:
* **Time travel**: Query historical versions of data
* **Rollback**: Restore tables to previous states
* **Audit trails**: Track all data changes
* **Reproducibility**: Ensure consistent results across experiments

## Versioning Strategy

### 1. Raw Data Versioning

**Tables:**
* `recomart.raw.user_interactions`
* `recomart.raw.transactions`
* `recomart.raw.products`

**Versioning Approach:**
* Each ingestion creates a new version (append mode for interactions/transactions)
* Products table uses overwrite mode (full refresh)
* Retention: 30 days of version history
* Metadata tracked: ingestion_timestamp, source_file, ingestion_batch_id

**Query Historical Versions:**
```sql
-- Query data as of specific timestamp
SELECT * FROM recomart.raw.user_interactions 
TIMESTAMP AS OF '2026-04-20 10:00:00';

-- Query data as of specific version
SELECT * FROM recomart.raw.user_interactions 
VERSION AS OF 5;

-- View version history
DESCRIBE HISTORY recomart.raw.user_interactions;
```

### 2. Processed Data Versioning

**Tables:**
* `recomart.processed.user_interactions_clean`
* `recomart.processed.transactions_clean`
* `recomart.processed.products_clean`

**Versioning Approach:**
* Each cleaning run creates new version
* Mode: overwrite (full refresh)
* Tracked: cleaning_timestamp, raw_version_used, records_before, records_after

**Rollback Example:**
```sql
-- Restore table to previous version
RESTORE TABLE recomart.processed.user_interactions_clean 
TO VERSION AS OF 3;

-- Or restore to specific timestamp
RESTORE TABLE recomart.processed.user_interactions_clean 
TO TIMESTAMP AS OF '2026-04-19 15:30:00';
```

### 3. Feature Store Versioning

**Tables:**
* `recomart.features.user_features`
* `recomart.features.item_features`
* `recomart.features.user_item_interactions`
* `recomart.features.feature_metadata`

**Versioning Approach:**
* Features versioned with timestamp and version_id
* Metadata table tracks: feature_name, version, creation_date, source_tables, source_versions
* Unity Catalog Feature Store provides native versioning
* Each model training run references specific feature versions

**Feature Versioning Code:**
```python
from databricks.feature_engineering import FeatureEngineeringClient

fe = FeatureEngineeringClient()

# Create versioned feature table
fe.create_table(
    name="recomart.features.user_features",
    primary_keys=["user_id"],
    df=user_features_df,
    description="User-level features v1.0"
)

# Access specific version for training
training_set = fe.create_training_set(
    df=training_data,
    feature_lookups=[
        FeatureLookup(
            table_name="recomart.features.user_features",
            lookup_key="user_id",
            timestamp_lookup_key="event_timestamp"  # Point-in-time lookup
        )
    ],
    label="rating"
)
```

### 4. Model Versioning

**Registry:** MLflow Model Registry

**Versioning Approach:**
* Each training run creates new model version
* Models tagged with: feature_versions, data_versions, metrics
* Stages: None → Staging → Production → Archived
* Lineage tracked: training data versions, feature versions, hyperparameters

**Model Lineage Example:**
```python
import mlflow

with mlflow.start_run(run_name="ALS_Recommender_v1"):
    # Log data versions
    mlflow.log_param("raw_data_version", "v20260420")
    mlflow.log_param("features_version", "v1.2")
    mlflow.log_param("user_features_delta_version", 7)
    mlflow.log_param("item_features_delta_version", 5)
    
    # Train and log model
    model = train_als_model(...)
    mlflow.spark.log_model(model, "als_model")
    
    # Log metrics
    mlflow.log_metric("precision_at_10", 0.28)
    mlflow.log_metric("recall_at_10", 0.42)
    mlflow.log_metric("ndcg_at_10", 0.52)
```

## Data Lineage Tracking

### Lineage Graph

```
Raw Data (v1) → Data Validation → Cleaned Data (v1) → Feature Engineering → Features (v1.0) → Model Training → Model (v1)
     ↓                                     ↓                                        ↓                                  ↓
Version History              Quality Reports                        Feature Metadata                      MLflow Registry
```

### Table Dependencies

**Forward Lineage (Downstream):**
* `recomart.raw.user_interactions` → `recomart.processed.user_interactions_clean` → `recomart.features.user_features`
* `recomart.raw.transactions` → `recomart.processed.transactions_clean` → `recomart.features.user_features`, `recomart.features.user_item_interactions`
* `recomart.raw.products` → `recomart.processed.products_clean` → `recomart.features.item_features`

**Backward Lineage (Upstream):**
* Model v1 ← Features (user v1.0, item v1.0) ← Cleaned Data (v3) ← Raw Data (v8)

### Querying Lineage

```python
# Get table history
history_df = spark.sql("DESCRIBE HISTORY recomart.raw.user_interactions")
display(history_df)

# Get detailed version info
version_info = spark.sql("""
    DESCRIBE DETAIL recomart.raw.user_interactions
""")
display(version_info)

# Audit trail - who changed what when
audit_df = spark.sql("""
    SELECT 
        version,
        timestamp,
        operation,
        operationParameters,
        userName,
        operationMetrics
    FROM (DESCRIBE HISTORY recomart.raw.user_interactions)
    ORDER BY version DESC
""")
display(audit_df)
```

## Retention & Cleanup

### Vacuum Old Versions

```sql
-- Remove files older than 30 days
VACUUM recomart.raw.user_interactions RETAIN 720 HOURS;

-- Preview files to be deleted (dry run)
VACUUM recomart.raw.user_interactions DRY RUN;
```

### Optimize Tables

```sql
-- Compact small files for better performance
OPTIMIZE recomart.raw.user_interactions;

-- Z-order by frequently filtered columns
OPTIMIZE recomart.raw.user_interactions 
ZORDER BY (user_id, interaction_date);
```

## Best Practices

1. **Version Everything**: Raw data, processed data, features, models
2. **Tag Versions**: Add meaningful descriptions and tags to versions
3. **Document Dependencies**: Maintain lineage documentation
4. **Automate Cleanup**: Regular VACUUM operations for old versions
5. **Point-in-Time Consistency**: Use timestamp-based lookups for features
6. **Reproducibility**: Always log data/feature versions in model training
7. **Rollback Plan**: Test rollback procedures regularly
8. **Audit Trail**: Enable and monitor operation logs

## Reproducibility Example

To reproduce model training results:

```python
# 1. Identify versions used in original training
original_run = mlflow.get_run(run_id="abc123")
raw_version = original_run.data.params["raw_data_version"]
feature_version = original_run.data.params["features_version"]

# 2. Load exact same data versions
user_interactions = spark.sql(f"""
    SELECT * FROM recomart.raw.user_interactions 
    VERSION AS OF {raw_version}
""")

# 3. Regenerate features or load versioned features
features = spark.table(f"recomart.features.user_features@v{feature_version}")

# 4. Retrain with same hyperparameters
model = train_model(features, params=original_run.data.params)

# Results should be identical (deterministic)
```

## Monitoring & Alerts

**Version Monitoring:**
* Alert if version count exceeds threshold (> 100 versions)
* Alert on failed VACUUM operations
* Monitor storage growth from old versions

**Lineage Monitoring:**
* Track downstream impact of upstream data changes
* Alert on breaking schema changes
* Validate referential integrity across versions

## Tools & Commands Reference

**Delta Lake Commands:**
```sql
DESCRIBE HISTORY table_name [LIMIT n]
DESCRIBE DETAIL table_name
RESTORE TABLE table_name TO VERSION AS OF version_number
RESTORE TABLE table_name TO TIMESTAMP AS OF 'timestamp_string'
VACUUM table_name [RETAIN num HOURS] [DRY RUN]
OPTIMIZE table_name [ZORDER BY (col1, col2, ...)]
```

**Python API:**
```python
from delta.tables import DeltaTable

# Get Delta table
dt = DeltaTable.forName(spark, "recomart.raw.user_interactions")

# View history
history = dt.history()

# Vacuum
dt.vacuum(retentionHours=720)

# Optimize
dt.optimize().executeCompaction()
dt.optimize().executeZOrderBy("user_id", "interaction_date")
```

## Summary

The RecoMart pipeline uses Delta Lake's native versioning capabilities to ensure:
* **Auditability**: Complete history of all data changes
* **Reproducibility**: Ability to recreate any past state
* **Rollback**: Quick recovery from data issues
* **Compliance**: Full audit trail for regulatory requirements
* **Experimentation**: Safe exploration of data changes

All data versions are tracked from raw ingestion through model deployment, enabling complete end-to-end lineage and reproducibility.

---
**Document Version:** 1.0  
**Last Updated:** April 2026  
**Author:** RecoMart Data Platform Team

# RecoMart Data Storage Structure Documentation

## Overview
This document describes the data storage architecture for the RecoMart recommendation pipeline, including file system organization, Unity Catalog structure, and data partitioning strategies.

## Storage Architecture

### 1. Unity Catalog Structure

**Catalog-Schema-Table Hierarchy:**
```
recomart (Catalog)
├── raw (Schema - Bronze Layer)
│   ├── user_interactions
│   ├── transactions
│   └── products
├── clean (Schema - Silver Layer) [To be created]
│   ├── user_interactions_clean
│   ├── transactions_clean
│   └── products_clean
└── features (Schema - Gold Layer) [To be created]
    ├── user_features
    ├── item_features
    └── user_item_features
```

### 2. Delta Lake Tables Configuration

#### Table: recomart.raw.user_interactions
- **Source**: Web and mobile clickstream logs (CSV)
- **Format**: Delta Lake
- **Partitioning**: Partitioned by `interaction_date` (daily partitions)
- **Schema**:
  - user_id (string) - NOT NULL
  - item_id (string) - NOT NULL
  - interaction_type (string) - click, view, add_to_cart, purchase
  - timestamp (string) - NOT NULL
  - session_id (string)
  - device_type (string) - mobile, web, tablet
  - ingestion_timestamp (timestamp) - Audit field
  - source_file (string) - Audit field
  - ingestion_batch_id (string) - Audit field
  - interaction_date (date) - Partition column
- **Data Retention**: 90 days (configurable)
- **Optimization**: Z-ORDER by (user_id, item_id)

#### Table: recomart.raw.transactions
- **Source**: Order management system (CSV)
- **Format**: Delta Lake
- **Partitioning**: Partitioned by `transaction_date` (daily partitions)
- **Schema**:
  - transaction_id (string) - PRIMARY KEY
  - user_id (string) - NOT NULL
  - item_id (string) - NOT NULL
  - quantity (integer)
  - price (double) - NOT NULL
  - rating (integer) - Range: 1-5
  - timestamp (string)
  - payment_method (string)
  - ingestion_timestamp (timestamp) - Audit field
  - source_file (string) - Audit field
  - ingestion_batch_id (string) - Audit field
  - transaction_date (date) - Partition column
- **Data Retention**: 2 years
- **Optimization**: Z-ORDER by (user_id, item_id)

#### Table: recomart.raw.products
- **Source**: Product catalog API (JSON)
- **Format**: Delta Lake
- **Partitioning**: None (slowly changing dimension)
- **Schema**:
  - item_id (string) - PRIMARY KEY
  - product_name (string) - NOT NULL
  - category (string) - NOT NULL
  - sub_category (string)
  - brand (string)
  - price (double) - NOT NULL
  - description (string)
  - popularity_score (double) - Range: 0-100
  - sentiment_score (double) - Range: 0-1
  - stock_status (string)
  - ingestion_timestamp (timestamp) - Audit field
  - source_file (string) - Audit field
  - ingestion_batch_id (string) - Audit field
- **Update Strategy**: Full refresh (OVERWRITE mode)
- **Data Retention**: Current + 30 days history

### 3. File System Structure

**Project Root**: `/Workspace/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/`

```
RecoMart_Recommendation_Pipeline/
├── 01_Problem_Formulation/
│   └── Problem_Formulation.md
├── 02_Data_Ingestion/
│   └── 01_Data_Ingestion.ipynb
├── 03_Data_Storage/
│   └── Storage_Structure_Documentation.md (this file)
├── 04_Data_Validation/
│   └── 02_Data_Validation_Profiling.ipynb
├── 05_Data_Preparation/
│   └── 03_Data_Preparation_EDA.ipynb
├── 06_Feature_Engineering/
│   └── 04_Feature_Engineering_Store.ipynb
├── 07_Feature_Store/
│   └── (merged with Feature Engineering)
├── 08_Data_Versioning/
│   └── Data_Versioning_Documentation.md
├── 09_Model_Training/
│   └── 05_Model_Training_Evaluation.ipynb
├── 10_Pipeline_Orchestration/
│   └── 06_Workflow_Orchestration.ipynb
├── data/
│   ├── raw/                         # Raw source data files
│   │   ├── user_interactions/       # CSV files
│   │   │   └── interactions_YYYY_MM.csv
│   │   ├── transactions/            # CSV files
│   │   │   └── transactions_YYYY_MM.csv
│   │   └── products/                # JSON files
│   │       └── products_catalog_YYYY_MM.json
│   ├── processed/                   # Cleaned/transformed data
│   │   ├── user_interactions_clean/
│   │   ├── transactions_clean/
│   │   └── products_clean/
│   └── features/                    # Feature store exports
│       ├── user_features/
│       ├── item_features/
│       └── user_item_features/
├── logs/
│   ├── ingestion.log
│   ├── validation.log
│   ├── preparation.log
│   └── training.log
├── documentation/
│   ├── data_quality_reports/
│   └── model_evaluation_reports/
└── config/
    ├── ingestion_config.yaml
    ├── validation_rules.yaml
    └── model_config.yaml
```

### 4. Partitioning Strategy

#### Date-Based Partitioning (User Interactions & Transactions)
- **Partition Column**: Date extracted from timestamp
- **Granularity**: Daily partitions (YYYY-MM-DD)
- **Benefits**:
  - Efficient queries with date filters
  - Easy data retention management (DROP old partitions)
  - Parallel processing per partition
  - Reduced scan costs

**Example Query Using Partition Pruning:**
```sql
SELECT * FROM recomart.raw.user_interactions
WHERE interaction_date BETWEEN '2026-03-01' AND '2026-03-31'
-- Only scans March 2026 partitions, ignoring all others
```

#### No Partitioning (Products)
- Master data with relatively small size (~500-5000 products)
- Full table scan is acceptable
- Frequent updates require flexible structure
- Uses OVERWRITE mode for each load

### 5. Data Retention Policies

| Table | Retention Period | Cleanup Strategy |
|-------|-----------------|------------------|
| user_interactions | 90 days | Drop partitions older than 90 days |
| transactions | 2 years | Drop partitions older than 2 years |
| products | Current + 30 days | Maintain using Delta time travel |

**Retention Script Example:**
```python
# Drop old partitions for user_interactions
spark.sql("""
    ALTER TABLE recomart.raw.user_interactions
    DROP IF EXISTS PARTITION (interaction_date < '2026-01-20')
""")
```

### 6. Delta Lake Time Travel

All tables support time travel queries:

```python
# Query data as of a specific timestamp
df = spark.read.format("delta")\
    .option("timestampAsOf", "2026-04-19 12:00:00")\
    .table("recomart.raw.transactions")

# Query data as of a specific version
df = spark.read.format("delta")\
    .option("versionAsOf", 5)\
    .table("recomart.raw.transactions")

# View table history
spark.sql("DESCRIBE HISTORY recomart.raw.transactions").show()
```

### 7. Storage Optimization

#### OPTIMIZE Command
Regularly run OPTIMIZE to compact small files:
```sql
OPTIMIZE recomart.raw.user_interactions
ZORDER BY (user_id, item_id);
```

#### VACUUM Command
Remove old files after retention period:
```sql
VACUUM recomart.raw.user_interactions RETAIN 168 HOURS;
```

### 8. Access Control

**Unity Catalog Permissions:**
- `recomart` catalog: Data Engineers (READ/WRITE)
- `recomart.raw` schema: Ingestion service account (WRITE)
- `recomart.clean` schema: Data Scientists (READ)
- `recomart.features` schema: ML Engineers (READ/WRITE)

### 9. Monitoring & Audit

**Audit Metadata in Every Table:**
- `ingestion_timestamp`: When data was loaded
- `source_file`: Origin file name
- `ingestion_batch_id`: Unique batch identifier (YYYYMMDD_HHMMSS)

**Monitoring Queries:**
```sql
-- Check latest ingestion timestamp
SELECT MAX(ingestion_timestamp) as latest_load
FROM recomart.raw.user_interactions;

-- Count records by source file
SELECT source_file, COUNT(*) as record_count
FROM recomart.raw.transactions
GROUP BY source_file;

-- Data freshness check
SELECT 
    DATEDIFF(NOW(), MAX(ingestion_timestamp)) as days_since_last_load
FROM recomart.raw.products;
```

### 10. Disaster Recovery

**Delta Lake Capabilities:**
- Automatic checkpointing every 10 commits
- Transaction log provides full audit trail
- Time travel enables point-in-time recovery
- RESTORE command for rollback:

```sql
RESTORE TABLE recomart.raw.transactions
TO VERSION AS OF 10;
```

### 11. Storage Costs Optimization

**Best Practices:**
1. **Partition pruning**: Always filter on partition columns
2. **File compaction**: Run OPTIMIZE weekly
3. **Old version cleanup**: Run VACUUM monthly
4. **Compression**: Delta uses Parquet with Snappy compression (default)
5. **Column pruning**: Select only needed columns

**Cost Breakdown (Estimated):**
- Raw data storage: ~0.75 GB (uncompressed) → ~0.3 GB (compressed)
- Delta transaction logs: ~5 MB
- Checkpoints: ~10 MB
- **Total storage**: ~315 MB for 10,670 records

---

## Configuration Files

### ingestion_config.yaml
```yaml
catalog: recomart
schema: raw
retry_config:
  max_retries: 3
  retry_delay: 2
  backoff_multiplier: 2
  
data_sources:
  user_interactions:
    path: /data/raw/user_interactions
    format: csv
    partition_by: interaction_date
  transactions:
    path: /data/raw/transactions
    format: csv
    partition_by: transaction_date
  products:
    path: /data/raw/products
    format: json
    partition_by: null
```

### validation_rules.yaml
```yaml
user_interactions:
  not_null: [user_id, item_id, timestamp]
  categorical:
    interaction_type: [click, view, add_to_cart, purchase]
    device_type: [mobile, web, tablet]
    
transactions:
  not_null: [transaction_id, user_id, item_id, price]
  unique: [transaction_id]
  ranges:
    rating: [1, 5]
    price: [0.01, 10000]
    quantity: [1, 100]
    
products:
  not_null: [item_id, product_name, category, price]
  unique: [item_id]
  ranges:
    price: [0.01, 10000]
    popularity_score: [0, 100]
    sentiment_score: [0, 1]
```

---

**Document Version**: 1.0  
**Last Updated**: 2026-04-20  
**Maintained By**: RecoMart Data Platform Team

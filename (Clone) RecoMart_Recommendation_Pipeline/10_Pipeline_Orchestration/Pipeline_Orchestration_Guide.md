# RecoMart Pipeline Orchestration with Databricks Workflows

## Overview
This document provides a comprehensive guide for orchestrating the end-to-end RecoMart recommendation pipeline using Databricks Workflows (formerly Databricks Jobs).

## Pipeline Architecture

### Complete Data Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                    RECOMART ML PIPELINE                          │
└─────────────────────────────────────────────────────────────────┘

[Raw Data Sources]
       ↓
[Task 1: Data Ingestion] ────→ Delta: recomart.raw.*
       ↓
[Task 2: Data Validation] ────→ Quality Reports
       ↓
[Task 3: Data Preparation] ───→ Delta: recomart.clean.*
       ↓
[Task 4: Feature Engineering] → Delta: recomart.features.*
       ↓
[Task 5: Model Training] ──────→ MLflow Model Registry
       ↓
[Task 6: Model Evaluation] ────→ Metrics & Reports
```

## Workflow Configuration

### Workflow Overview
**Name**: `RecoMart_Daily_ML_Pipeline`  
**Schedule**: Daily at 2:00 AM UTC  
**Timeout**: 2 hours  
**Retries**: 2 attempts per task  
**Notifications**: Email on failure  

### Task Dependencies

```
Task 1 (Ingestion)
    ↓
Task 2 (Validation) ──→ Task 3 (Preparation)
                            ↓
                        Task 4 (Features)
                            ↓
                        Task 5 (Training)
                            ↓
                        Task 6 (Evaluation)
```

## Task Definitions

### Task 1: Data Ingestion
**Notebook**: `02_Data_Ingestion/01_Data_Ingestion`  
**Cluster**: Shared compute (4-8 workers)  
**Timeout**: 30 minutes  
**Retry**: 2 times, 5-minute delay  

**Purpose**: Ingest raw data from CSV/JSON sources into Delta tables

**Parameters**:
```json
{
  "ingestion_date": "{{job.start_time.iso_date}}",
  "source_path": "/data/raw"
}
```

**Success Criteria**:
* All 3 data sources ingested successfully
* Record counts > 0 for each source
* No critical errors in logs

---

### Task 2: Data Validation
**Notebook**: `04_Data_Validation/02_Data_Validation_Profiling`  
**Depends On**: Task 1 (Ingestion)  
**Cluster**: Shared compute (4-8 workers)  
**Timeout**: 20 minutes  

**Purpose**: Validate data quality and generate quality reports

**Quality Gates**:
* Overall quality score > 70
* Missing values < 10% in critical fields
* Duplicate rate < 5%

**Outputs**:
* Quality report JSON
* Flagged records for review

**Failure Behavior**:
* If quality score < 70: Send alert but continue pipeline
* If critical failures: Stop pipeline

---

### Task 3: Data Preparation & EDA
**Notebook**: `05_Data_Preparation/03_Data_Preparation_EDA`  
**Depends On**: Task 2 (Validation)  
**Cluster**: Shared compute (8-16 workers)  
**Timeout**: 45 minutes  

**Purpose**: Clean data, handle missing values, remove outliers

**Parameters**:
```json
{
  "remove_duplicates": true,
  "handle_nulls": "drop",
  "outlier_method": "IQR"
}
```

**Outputs**:
* `recomart.clean.user_interactions_clean`
* `recomart.clean.transactions_clean`
* `recomart.clean.products_clean`
* EDA visualizations

---

### Task 4: Feature Engineering
**Notebook**: `06_Feature_Engineering/04_Feature_Engineering_Store`  
**Depends On**: Task 3 (Preparation)  
**Cluster**: Compute-optimized (8-16 workers)  
**Timeout**: 1 hour  

**Purpose**: Compute user, item, and user-item features

**Parameters**:
```json
{
  "reference_date": "{{job.start_time.iso_date}}",
  "lookback_days": 90,
  "min_interactions": 5
}
```

**Outputs**:
* `recomart.features.user_features`
* `recomart.features.item_features`
* `recomart.features.user_item_features`

---

### Task 5: Model Training
**Notebook**: `09_Model_Training/05_Model_Training_Evaluation`  
**Depends On**: Task 4 (Feature Engineering)  
**Cluster**: ML Runtime (GPU-enabled, optional)  
**Timeout**: 1 hour  

**Purpose**: Train ALS recommendation model

**Parameters**:
```json
{
  "model_type": "ALS",
  "rank": 10,
  "max_iter": 10,
  "reg_param": 0.1,
  "test_size": 0.2,
  "random_seed": 42
}
```

**MLflow Integration**:
* Experiment: `/RecoMart_Recommendation_Experiment`
* Log parameters, metrics, and model artifacts
* Register model to MLflow Model Registry

**Outputs**:
* Trained ALS model in MLflow
* Evaluation metrics (RMSE, Precision@K, Recall@K)
* Model documentation

---

### Task 6: Model Evaluation & Deployment (Optional)
**Notebook**: Custom deployment notebook  
**Depends On**: Task 5 (Training)  
**Cluster**: Shared compute  
**Timeout**: 15 minutes  

**Purpose**: Evaluate model against production baseline and deploy if improved

**Evaluation Criteria**:
* Precision@10 > baseline + 5%
* Recall@10 > baseline + 5%
* Coverage > 80%

**Deployment Actions**:
* If criteria met: Promote model to Production stage
* Update model serving endpoint
* Send success notification

---

## Databricks Workflow Setup

### Method 1: Create via Databricks UI

#### Step 1: Navigate to Workflows
1. Click **Workflows** in the left sidebar
2. Click **Create Job**
3. Enter job name: `RecoMart_Daily_ML_Pipeline`

#### Step 2: Add Tasks

**Task 1 - Data Ingestion**:
* Click **Add task**
* Task name: `01_data_ingestion`
* Type: **Notebook**
* Source: Select `02_Data_Ingestion/01_Data_Ingestion`
* Cluster: Select or create shared cluster
* Timeout: 30 minutes
* Retries: 2
* Click **Create**

**Task 2 - Data Validation**:
* Click **Add task**
* Task name: `02_data_validation`
* Type: **Notebook**
* Source: Select `04_Data_Validation/02_Data_Validation_Profiling`
* **Depends on**: Select `01_data_ingestion`
* Cluster: Same as Task 1
* Click **Create**

**Task 3 - Data Preparation**:
* Click **Add task**
* Task name: `03_data_preparation`
* Type: **Notebook**
* Source: Select `05_Data_Preparation/03_Data_Preparation_EDA`
* **Depends on**: Select `02_data_validation`
* Cluster: Larger cluster (8-16 workers)
* Click **Create**

**Task 4 - Feature Engineering**:
* Click **Add task**
* Task name: `04_feature_engineering`
* Type: **Notebook**
* Source: Select `06_Feature_Engineering/04_Feature_Engineering_Store`
* **Depends on**: Select `03_data_preparation`
* Cluster: Same as Task 3
* Click **Create**

**Task 5 - Model Training**:
* Click **Add task**
* Task name: `05_model_training`
* Type: **Notebook**
* Source: Select `09_Model_Training/05_Model_Training_Evaluation`
* **Depends on**: Select `04_feature_engineering`
* Cluster: ML Runtime cluster
* Click **Create**

#### Step 3: Configure Schedule
1. Click **Add trigger** (top right)
2. Select **Scheduled**
3. Set schedule: **Cron expression**
   ```
   0 2 * * * ?
   ```
   (Daily at 2:00 AM UTC)
4. Timezone: **UTC**
5. Click **Save**

#### Step 4: Configure Notifications
1. Click **Edit** next to Email notifications
2. Add emails for:
   * **On start**: Optional
   * **On success**: data-team@recomart.com
   * **On failure**: data-team@recomart.com, sre-team@recomart.com
3. Click **Save**

#### Step 5: Set Job Parameters (Optional)
1. Click **Edit** next to Parameters
2. Add key-value pairs:
   ```
   environment: production
   data_path: /data/raw
   min_quality_score: 70
   ```

#### Step 6: Test the Workflow
1. Click **Run now** (top right)
2. Monitor execution in the Runs tab
3. Check logs for each task
4. Verify outputs in Delta tables

---

### Method 2: Create via Databricks CLI

#### Prerequisites
```bash
# Install Databricks CLI
pip install databricks-cli

# Configure authentication
databricks configure --token
# Enter workspace URL and personal access token
```

#### Create Job Definition File

Create `recomart_pipeline_job.json`:

```json
{
  "name": "RecoMart_Daily_ML_Pipeline",
  "email_notifications": {
    "on_failure": ["data-team@recomart.com"],
    "on_success": ["data-team@recomart.com"]
  },
  "timeout_seconds": 7200,
  "max_concurrent_runs": 1,
  "schedule": {
    "quartz_cron_expression": "0 0 2 * * ?",
    "timezone_id": "UTC"
  },
  "tasks": [
    {
      "task_key": "ingestion",
      "notebook_task": {
        "notebook_path": "/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/02_Data_Ingestion/01_Data_Ingestion",
        "source": "WORKSPACE"
      },
      "new_cluster": {
        "spark_version": "14.3.x-scala2.12",
        "node_type_id": "i3.xlarge",
        "num_workers": 4
      },
      "timeout_seconds": 1800,
      "max_retries": 2,
      "min_retry_interval_millis": 300000
    },
    {
      "task_key": "validation",
      "depends_on": [{"task_key": "ingestion"}],
      "notebook_task": {
        "notebook_path": "/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/04_Data_Validation/02_Data_Validation_Profiling",
        "source": "WORKSPACE"
      },
      "new_cluster": {
        "spark_version": "14.3.x-scala2.12",
        "node_type_id": "i3.xlarge",
        "num_workers": 4
      },
      "timeout_seconds": 1200
    },
    {
      "task_key": "preparation",
      "depends_on": [{"task_key": "validation"}],
      "notebook_task": {
        "notebook_path": "/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/05_Data_Preparation/03_Data_Preparation_EDA",
        "source": "WORKSPACE"
      },
      "new_cluster": {
        "spark_version": "14.3.x-scala2.12",
        "node_type_id": "i3.xlarge",
        "num_workers": 8
      },
      "timeout_seconds": 2700
    },
    {
      "task_key": "features",
      "depends_on": [{"task_key": "preparation"}],
      "notebook_task": {
        "notebook_path": "/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/06_Feature_Engineering/04_Feature_Engineering_Store",
        "source": "WORKSPACE"
      },
      "new_cluster": {
        "spark_version": "14.3.x-scala2.12",
        "node_type_id": "i3.xlarge",
        "num_workers": 8
      },
      "timeout_seconds": 3600
    },
    {
      "task_key": "training",
      "depends_on": [{"task_key": "features"}],
      "notebook_task": {
        "notebook_path": "/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/09_Model_Training/05_Model_Training_Evaluation",
        "source": "WORKSPACE"
      },
      "new_cluster": {
        "spark_version": "14.3.x-cpu-ml-scala2.12",
        "node_type_id": "i3.xlarge",
        "num_workers": 8
      },
      "timeout_seconds": 3600
    }
  ]
}
```

#### Create the Job
```bash
databricks jobs create --json-file recomart_pipeline_job.json
```

#### Run the Job
```bash
# Get job ID from output of create command
databricks jobs run-now --job-id <JOB_ID>
```

---

## Monitoring & Logging

### Databricks UI Monitoring
1. Navigate to **Workflows** → **Job runs**
2. View:
   * Run duration
   * Task-level success/failure
   * Logs for each task
   * Gantt chart of task execution
   * Spark UI for performance analysis

### Log Aggregation
All pipeline logs are written to:
```
/Workspace/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/logs/
  ├── ingestion.log
  ├── validation.log
  ├── preparation.log
  ├── features.log
  └── training.log
```

Access logs programmatically:
```python
# Read ingestion logs
with open('/Workspace/.../logs/ingestion.log', 'r') as f:
    logs = f.read()
    print(logs)
```

### MLflow Experiment Tracking
View all model training runs:
1. Navigate to **Machine Learning** → **Experiments**
2. Open **RecoMart_Recommendation_Experiment**
3. Compare runs, metrics, and parameters

### Custom Monitoring Dashboard
Create a monitoring notebook that queries job run history:

```python
# Get recent job runs
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# List recent runs
runs = w.jobs.list_runs(job_id=<JOB_ID>, limit=10)

for run in runs.runs:
    print(f"Run {run.run_id}: {run.state.life_cycle_state}")
    print(f"  Start: {run.start_time}")
    print(f"  Duration: {run.execution_duration}ms")
```

---

## Failure Handling & Alerts

### Automatic Retries
Each task configured with:
* **Max retries**: 2 attempts
* **Retry interval**: 5 minutes
* **Strategy**: Exponential backoff

### Email Alerts
Configured for:
* Pipeline start (optional)
* Pipeline success
* Pipeline failure
* Task failure

### Slack Integration (Optional)
```python
# In each notebook, add Slack webhook for notifications
import requests

def send_slack_alert(message, webhook_url):
    payload = {"text": message}
    requests.post(webhook_url, json=payload)

# On failure
try:
    # ... pipeline code ...
except Exception as e:
    send_slack_alert(
        f"🚨 RecoMart Pipeline Failed: {str(e)}",
        "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    )
    raise
```

### PagerDuty Integration
For critical failures, integrate with PagerDuty:
```python
import pdpyras

session = pdpyras.EventsAPISession('YOUR_INTEGRATION_KEY')

session.trigger('Pipeline failure', 'critical', 
                dedup_key='recomart-pipeline',
                custom_details={'error': str(e)})
```

---

## Performance Optimization

### Cluster Sizing Guidelines
| Task | Recommended Config | Rationale |
|------|-------------------|-----------|
| Ingestion | 4-8 workers, i3.xlarge | I/O bound, moderate compute |
| Validation | 4-8 workers, i3.xlarge | Lightweight validation logic |
| Preparation | 8-16 workers, i3.xlarge | Data transformation heavy |
| Features | 8-16 workers, i3.xlarge | Complex aggregations |
| Training | 8-16 workers, ML Runtime | Model training compute |

### Optimization Strategies
1. **Enable Autoscaling**: Set min/max workers for cost efficiency
2. **Use Spot Instances**: 60-80% cost savings for fault-tolerant tasks
3. **Cluster Pooling**: Reduce startup time with instance pools
4. **Photon Acceleration**: Enable Photon for SQL-heavy tasks

### Monitoring Costs
```sql
-- Query job run costs
SELECT 
    run_id,
    run_name,
    cluster_instance,
    execution_duration_ms / 3600000.0 AS duration_hours,
    duration_hours * cluster_hourly_cost AS estimated_cost
FROM system.billing.usage
WHERE workspace_id = current_workspace_id()
  AND job_name = 'RecoMart_Daily_ML_Pipeline'
  AND date > current_date() - 30
ORDER BY date DESC
```

---

## Deployment Checklist

Before deploying to production:

- [ ] All notebooks tested individually
- [ ] Task dependencies validated
- [ ] Cluster configurations optimized
- [ ] Timeout values appropriate
- [ ] Retry logic configured
- [ ] Email notifications set up
- [ ] Schedule configured correctly (timezone!)
- [ ] Access permissions granted (service principals)
- [ ] Monitoring dashboard created
- [ ] Runbook documented for failure scenarios
- [ ] Cost estimates approved
- [ ] Backup/rollback plan documented

---

## Demo Video Script

For the assignment demo video (5-10 minutes):

### Segment 1: Pipeline Overview (1 min)
* Show the Workflows UI
* Explain the 6 tasks and dependencies
* Highlight the DAG visualization

### Segment 2: Task Configuration (2 mins)
* Show one task configuration in detail
* Explain notebooks, clusters, retries
* Show parameter passing between tasks

### Segment 3: Run the Pipeline (3 mins)
* Click "Run now"
* Show real-time execution
* Navigate through task logs
* Show Delta tables being populated

### Segment 4: Monitoring (2 mins)
* Show MLflow experiment with metrics
* Display feature tables in Unity Catalog
* Show data lineage in Catalog Explorer

### Segment 5: Results (2 mins)
* Query final model from MLflow
* Show Precision@K, Recall@K metrics
* Demonstrate model inference

---

**Document Version**: 1.0  
**Last Updated**: 2026-04-20  
**Maintained By**: RecoMart Data Platform Team

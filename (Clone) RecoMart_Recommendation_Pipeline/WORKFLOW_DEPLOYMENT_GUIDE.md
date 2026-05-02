# RecoMart Workflow - Quick Deployment Guide

## ✅ Status: Configuration Ready

**Configuration File Location:**
`/Workspace/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/recomart_pipeline_job.json`

---

## 🚀 Deploy via Databricks UI (5 minutes)

### Step-by-Step:

1. **Navigate to Workflows**
   * Click `Workflows` in left sidebar → `Create Job`

2. **Basic Settings**
   * Job Name: `RecoMart_Daily_ML_Pipeline`
   * Timeout: `7200` seconds (2 hours)
   * Max Concurrent Runs: `1`

3. **Add 5 Tasks (in order):**

   | # | Task Key | Notebook Path | Timeout | Depends On |
   |---|----------|--------------|---------|------------|
   | 1 | ingestion | 02_Data_Ingestion/01_Data_Ingestion | 1800s | (none) |
   | 2 | validation | 04_Data_Validation/02_Data_Validation_Profiling | 1200s | ingestion |
   | 3 | preparation | 05_Data_Preparation/03_Data_Preparation_EDA | 2700s | validation |
   | 4 | features | 06_Feature_Engineering/04_Feature_Engineering_Store | 3600s | preparation |
   | 5 | training | 09_Model_Training/05_Model_Training_Evaluation | 3600s | features |

   **For each task:**
   * Click `Add task`
   * Enter task key name
   * Select `Notebook` type
   * Browse to notebook path
   * Set timeout and retries (2)
   * Select dependency (if applicable)

4. **Configure Schedule**
   * Click `Add trigger` → `Scheduled`
   * Cron: `0 0 2 * * ?`
   * Timezone: `UTC`
   * Status: `Active`

5. **Email Notifications**
   * On Success: `2025ae05415@wilp.bits-pilani.ac.in`
   * On Failure: `2025ae05415@wilp.bits-pilani.ac.in`

6. **Save & Test**
   * Click `Create`
   * Click `Run now` to test

---

## 🎯 What Happens When It Runs?

```
START → Ingestion (30m)
          ↓
       Validation (20m) ✓ Quality checks
          ↓
       Preparation (45m) ✓ Data cleaning
          ↓
       Features (60m) ✓ Feature engineering
          ↓
       Training (60m) ✓ ALS model + MLflow
          ↓
        END → Email notification
```

**Total Runtime:** ~3-4 hours (first run), ~1-2 hours (subsequent runs)

---

## 📊 Monitoring After Deployment

### View Job Runs:
1. `Workflows` → `RecoMart_Daily_ML_Pipeline`
2. Click `Runs` tab
3. View Gantt chart, logs, metrics

### Check MLflow Models:
1. `Machine Learning` → `Experiments`
2. Open `RecoMart_Recommendation_Experiment`
3. View metrics, compare runs

### Query Delta Tables:
```sql
-- Check latest ingestion
SELECT COUNT(*) FROM recomart.raw.user_interactions;

-- Check features
SELECT COUNT(*) FROM recomart.features.user_features;

-- View model metrics from training logs
```

---

## 🎬 Demo Video Checklist

**Show in your recording:**
1. ✅ Workflow configuration in UI (DAG view)
2. ✅ Click "Run now" and show execution
3. ✅ Task-by-task progression (Gantt chart)
4. ✅ Navigate to MLflow to show logged model
5. ✅ Query final Delta tables in SQL
6. ✅ Show Precision@K, Recall@K metrics

**Estimated demo time:** 5-10 minutes

---

## 📝 Assignment Submission Checklist

- [ ] Workflow created and tested (screenshot)
- [ ] All 5 notebooks run successfully
- [ ] MLflow model registered (show experiment)
- [ ] Delta tables populated (row counts)
- [ ] Demo video recorded (5-10 min)
- [ ] All documentation files included
- [ ] .zip file prepared for submission

---

**Created:** 2026-04-21  
**Pipeline:** RecoMart ML Recommendation System  
**Status:** Ready for Production Deployment

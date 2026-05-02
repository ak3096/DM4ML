# RecoMart ALS Recommendation Model

## Model Information
* **Model Type**: Collaborative Filtering (ALS - Alternating Least Squares)
* **Training Date**: 2026-04-20 19:32:02
* **Status**: Local Training Only

## Hyperparameters
* Rank (Latent Factors): 10
* Max Iterations: 10
* Regularization: 0.1
* Alpha (Confidence): 1.0

## Training Data
* Training Set: 3,749 records
* Test Set: 955 records

## Performance Metrics
* **RMSE**: 3.3389
* **MAE**: 3.0169
* **Precision@10**: 0.0026
* **Recall@10**: 0.0169
* **F1@10**: 0.0046
* **Coverage**: 50.20%

## Model Usage

### Generate Recommendations for a User
```python
# Get recommendations for user_idx
user_recommendations = als_model.recommendForUserSubset(
    spark.createDataFrame([(user_idx,)], ["user_idx"]),
    num_recommendations=10
)
```

### Generate Recommendations for All Users
```python
all_recommendations = als_model.recommendForAllUsers(10)
```

### Get Similar Items
```python
item_recommendations = als_model.recommendForItemSubset(
    spark.createDataFrame([(item_idx,)], ["item_idx"]),
    num_recommendations=10
)
```

## Files Generated
* Metrics: /Workspace/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/logs/model_metrics_20260420_193202.json
* Logs: /Workspace/Users/2025ae05415@wilp.bits-pilani.ac.in/RecoMart_Recommendation_Pipeline/logs/training.log

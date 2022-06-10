# Credit-card-fraud-detection
A real time credit card fraud detection using Google Cloud.
**Created by: Ravali Attivilli**

## **Pre-requisites:** 
The following roles and permisisons must be enabled in IAM-

```
1) roles/pubsub.editor
2) roles/storage.admin
3) roles/bigquery.dataEditor
4) roles/bigquery.jobUser
5) roles/ml.developer
6) roles/datastore.user
7) roles/dataflow.developer
8) roles/compute.viewer
```

## **Create a Pub Sub topic:**
```
gcloud pubsub topics create $PUBSUB_TOPIC_NAME
gcloud pubsub subscriptions create $PUBSUB_SUBSCRIPTION_NAME --topic=$PUBSUB_TOPIC_NAME

```

## **Create a GCS bucket topic:**
Used to export the model to serve staging directories for the dataflow pipeline
```
gsutil mb gs://$BUCKET_NAME

```

## **Create a Pub/Sub topic:**
Acts as a fraud notification channel
```
gcloud pubsub topics create $PUBSUB_NOTIFICATION_TOPIC

```
## **Create the output BQ dataset:**
Used to hold transactions, predictions and confidence scores
```
bq --location=US-central1 mk --dataset $PROJECT_ID:$DATASET
bq mk --table $DATASET.$OUTPUT_BQ_TABLE utilities/output_schema.json

```
## BigQuery View definitions
The queries used for creating the publicly available views are provided below for reference.
### Training Data for simple model
```
CREATE VIEW `extreme-lore-352210.cc_data.train_simple`  AS (
SELECT 
      EXTRACT (dayofweek FROM trans_date_trans_time) AS day,
      DATE_DIFF(EXTRACT(DATE FROM trans_date_trans_time),dob, YEAR) AS age,
      ST_DISTANCE(ST_GEOGPOINT(long,lat), ST_GEOGPOINT(merch_long, merch_lat)) AS distance,
      category,
      amt,
      state,
      job,
      unix_time,
      city_pop,
      merchant,
      is_fraud
  FROM (
    SELECT * EXCEPT(cc_num)
    FROM `extreme-lore-352210.cc_data.train_raw`  t1
    LEFT JOIN `extreme-lore-352210.cc_data.demographics` t2 ON t1.cc_num =t2.cc_num))
```
### Training Data for model with aggregates
```
CREATE VIEW `extreme-lore-352210.cc_data.train_w_aggregates`  AS (
SELECT  
    EXTRACT (dayofweek FROM trans_date_trans_time) AS day,
    DATE_DIFF(EXTRACT(DATE FROM trans_date_trans_time),dob, YEAR) AS age,
    ST_DISTANCE(ST_GEOGPOINT(long,lat), ST_GEOGPOINT(merch_long, merch_lat)) AS distance,
    TIMESTAMP_DIFF(trans_date_trans_time, last_txn_date , MINUTE) AS trans_diff, 
    AVG(amt) OVER(
                PARTITION BY cc_num
                ORDER BY unix_time
                -- 1 week is 604800 seconds
                RANGE BETWEEN 604800 PRECEDING AND 1 PRECEDING) AS avg_spend_pw,
    AVG(amt) OVER(
                PARTITION BY cc_num
                ORDER BY unix_time
                -- 1 month(30 days) is 2592000 seconds
                RANGE BETWEEN 2592000 PRECEDING AND 1 PRECEDING) AS avg_spend_pm,
    COUNT(*) OVER(
                PARTITION BY cc_num
                ORDER BY unix_time
                -- 1 day is 86400 seconds
                RANGE BETWEEN 86400 PRECEDING AND 1 PRECEDING ) AS trans_freq_24,
    category,
    amt,
    state,
    job,
    unix_time,
    city_pop,
    merchant,
    is_fraud
  FROM (
          SELECT t1.*,t2.* EXCEPT(cc_num),
              LAG(trans_date_trans_time) OVER (PARTITION BY t1.cc_num ORDER BY trans_date_trans_time ASC) AS last_txn_date,
          FROM  `extreme-lore-352210.cc_data.train_raw`  t1
          LEFT JOIN  `extreme-lore-352210.cc_data.demographics`  t2 ON t1.cc_num =t2.cc_num))
```
### Testing Data for simple model
```
CREATE VIEW `extreme-lore-352210.cc_data.test_simple` AS (
SELECT 
      EXTRACT (dayofweek FROM trans_date_trans_time) AS day,
      DATE_DIFF(EXTRACT(DATE FROM trans_date_trans_time),dob, YEAR) AS age,
      ST_DISTANCE(ST_GEOGPOINT(long,lat), ST_GEOGPOINT(merch_long, merch_lat)) AS distance,
      category,
      amt,
      state,
      job,
      unix_time,
      city_pop,
      merchant,
      is_fraud
  FROM (
    SELECT * EXCEPT(cc_num)
    FROM `extreme-lore-352210.cc_data.test_raw` t1
    LEFT JOIN `extreme-lore-352210.cc_data.demographics` t2 ON t1.cc_num =t2.cc_num))
```
### Testing Data for model with aggregates
```
CREATE VIEW `extreme-lore-352210.cc_data.test_w_aggregates`  AS (
WITH t1 as (
SELECT *, 'train' AS split FROM  `extreme-lore-352210.cc_data.train_raw` 
UNION ALL 
SELECT *, 'test' AS split FROM  `extreme-lore-352210.cc_data.test_raw`
),
v2 AS (
  SELECT t1.*,t2.* EXCEPT(cc_num),
              LAG(trans_date_trans_time) OVER (PARTITION BY t1.cc_num ORDER BY trans_date_trans_time ASC) AS last_txn_date,
            FROM t1
            LEFT JOIN  `extreme-lore-352210.cc_data.demographics` t2 ON t1.cc_num = t2.cc_num
),
v3 AS (
  SELECT
        EXTRACT (dayofweek FROM trans_date_trans_time) as day,
        DATE_DIFF(EXTRACT(DATE FROM trans_date_trans_time),dob, YEAR) AS age,
        ST_DISTANCE(ST_GEOGPOINT(long,lat), ST_GEOGPOINT(merch_long, merch_lat)) as distance,
        TIMESTAMP_DIFF(trans_date_trans_time, last_txn_date , MINUTE) AS trans_diff, 
        Avg(amt) OVER(
                    PARTITION BY cc_num
                    ORDER BY unix_time
                    RANGE BETWEEN 604800 PRECEDING AND 1 PRECEDING) AS avg_spend_pw,
        Avg(amt) OVER(
                    PARTITION BY cc_num
                    ORDER BY unix_time
                    RANGE BETWEEN 2592000 PRECEDING AND 1 PRECEDING) AS avg_spend_pm,
        count(*) OVER(
                    PARTITION BY cc_num
                    ORDER BY unix_time
                    RANGE BETWEEN 86400 PRECEDING AND 1 PRECEDING) AS trans_freq_24,
        category,
        amt,
        state,
        job,
        unix_time,
        city_pop,
        merchant,
        is_fraud,
        split
    FROM v2
)
SELECT * EXCEPT(split) FROM v3 WHERE split='test')
```
## ML Model Training
Do substitution for variables PROJECT_ID, DATASET and MODEL_NAME_WITHOUT_AGG
### Model 1 (without aggregates)
```
CREATE OR REPLACE MODEL 
  `[PROJECT_ID].[DATASET].[MODEL_NAME_WITHOUT_AGG]`
OPTIONS (
  model_type ='BOOSTED_TREE_CLASSIFIER',
  NUM_PARALLEL_TREE =8,
  MAX_ITERATIONS = 50,
  input_label_cols=["is_fraud"]
) AS
SELECT 
  *
FROM 
  `extreme-lore-352210.cc_data.train_simple`
```


### Model 2 (with aggregates)
```
CREATE OR REPLACE MODEL 
  `[PROJECT_ID].[DATASET].[MODEL_NAME_WITH_AGG]`
OPTIONS (
  model_type ='BOOSTED_TREE_CLASSIFIER',
  NUM_PARALLEL_TREE =8,
  MAX_ITERATIONS = 50,
  input_label_cols=["is_fraud"]
) AS
SELECT
  *
FROM 
  `extreme-lore-352210.cc_data.train_w_aggregates`
```
         

## Model Evaluation
Test both the models using ML.EVALUATE
```
SELECT 
  "simplemodel" AS model_name,
  *
FROM
  ML.EVALUATE(
    MODEL `[PROJECT_ID].[DATASET].[MODEL_NAME_WITHOUT_AGG]`,
    (SELECT * FROM `extreme-lore-352210.cc_data.test_simple`))
UNION ALL
SELECT
  "model_w_aggregates" AS model_name,
  *
FROM
  ML.EVALUATE(
    MODEL `[PROJECT_ID].[DATASET].[MODEL_NAME_WITH_AGG]`,
    (SELECT * FROM  `extreme-lore-352210.cc_data.test_w_aggregates`))
```

## Exporting model to AI Platform

### Exporting model to GCS
1. Model 1 (without aggregates)
```
bq extract --destination_format ML_XGBOOST_BOOSTER -m $PROJECT_ID:$DATASET.$MODEL_NAME_WITHOUT_AGG gs://$BUCKET_NAME/$MODEL_NAME_WITHOUT_AGG
```
2. Model 2 (with aggregates)
```
bq extract --destination_format ML_XGBOOST_BOOSTER -m  $PROJECT_ID:$DATASET.$MODEL_NAME_WITH_AGG gs://$BUCKET_NAME/$MODEL_NAME_WITH_AGG
```
            

### Creating Model in AI Platform
1. Create a model in AI platform to hold both our versions (without aggregates and with aggregates)
```
gcloud ai-platform models create $AI_MODEL_NAME --region global
```
2. Create 1 model version for model without aggregates
```
gcloud beta ai-platform versions create $VERSION_NAME_WITHOUT_AGG \
--model=$AI_MODEL_NAME \
--origin=gs://$BUCKET_NAME/$MODEL_NAME_WITHOUT_AGG \
--package-uris=gs://$BUCKET_NAME/$MODEL_NAME_WITHOUT_AGG/xgboost_predictor-0.1.tar.gz \
--prediction-class=predictor.Predictor \
--runtime-version=1.15 \
--region global
```
3. Create 1 model version for model with aggregates
```
gcloud beta ai-platform versions create $VERSION_NAME_WITH_AGG \
--model=$AI_MODEL_NAME \
--origin=gs://$BUCKET_NAME/$MODEL_NAME_WITH_AGG \
--package-uris=gs://$BUCKET_NAME/$MODEL_NAME_WITH_AGG/xgboost_predictor-0.1.tar.gz \
--prediction-class=predictor.Predictor \
--runtime-version=1.15 \
--region global
```
### Testing Online Predictions
To test the deployed models, send online prediction requests using the sample inputs [input_wo_aggregates.json](sample_inputs/input_wo_aggregates.json) and [input_w_aggregates.json](sample_inputs/input_w_aggregates.json)
1. Model 1 (WITHOUT AGGREGATES)
```
gcloud ai-platform predict --model $AI_MODEL_NAME \
--version $VERSION_NAME_WITHOUT_AGG \
--region global \
--json-instances sample_inputs/input_wo_aggregates.json
```

2. Model 2 (WITH AGGREGATES)
```
gcloud ai-platform predict --model $AI_MODEL_NAME \
--version $VERSION_NAME_WITH_AGG \
--region global \
--json-instances sample_inputs/input_w_aggregates.json
```
### Load transaction history into Firestore
The past transaction history (from train_raw and test_raw BQ tables) needs to be loaded into Firestore so the dataflow pipeline can do lookup while the simulated transactions (simulation_data BQ table) is used for real time inference. Lauch the script [load_to_firestore.py](utilities/load_to_firestore.py) to fetch the train and test transactions from BigQuery and to load these documents into Firestore. Ensure you have google-cloud-bigquery and google-cloud-firestore python modules installed.
```
python3 utilities/load_to_firestore.py $PROJECT_ID $FIRESTORE_COL_NAME
```
### Launch the Dataflow pipeline
Run the below command to launch a streaming dataflow pipeline for ML inference
```
python3 inference_pipeline.py \
--project $PROJECT_ID \
--firestore-project $PROJECT_ID \
--subscription-name $PUBSUB_SUBSCRIPTION_NAME \
--firestore-collection $FIRESTORE_COL_NAME \
--dataset-id $DATASET \
--table-name $OUTPUT_BQ_TABLE \
--model-name $AI_MODEL_NAME \
--model-with-aggregates $VERSION_NAME_WITH_AGG \
--model-without-aggregates $VERSION_NAME_WITHOUT_AGG \
--fraud-notification-topic $PUBSUB_NOTIFICATION_TOPIC \
--staging_location gs://$BUCKET_NAME/dataflow/staging \
--temp_location gs://$BUCKET_NAME/dataflow/tmp/ \
--region us-east1 \
--streaming \
--runner DataflowRunner \
--job_name predict-fraudulent-transactions \
--requirements_file requirements.txt
```

### Simulate real time transactions
Launch the python script [bq_to_pubsub.py](utilities/bq_to_pubsub.py) which reads from BigQuery table simulation_data and ingests records into input pubsub topic to simulate real time traffic on the Dataflow pipeline

```
python3 utilities/bq_to_pubsub.py $PROJECT_ID $PUBSUB_TOPIC_NAME
```

For production grade pipelines, one need to consider AI Platform quotas such as [number of Online Predictions per minute](https://cloud.google.com/ai-platform/prediction/docs/quotas#online_prediction_requests)
  
### Monitoring Dashboard
We have created a sample dashboard to get started on monitoring few of the important metrics, refer to [dashboard template](utilities/dashboard_template.json). Replace your project ID or number in the template to create the dashboard in your project.
  

### Clean Up
To avoid incurring charges to your Google Cloud account, ensure you terminate the resources used in this pattern




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
```gsutil mb gs://$BUCKET_NAME
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


# Serverless Data Pipeline on GCP
Useful link(s):
* [How to Schedule a Recurring Python Script on GCP](https://cloud.google.com/blog/products/application-development/how-to-schedule-a-recurring-python-script-on-gcp)
* [How to use GCP service account in Google Cloud Function? - Stack Overflow](https://stackoverflow.com/questions/55671256/how-to-use-gcp-service-account-in-google-cloud-function)
* [python - How to handle keys and credentials when deploying to Google Cloud Functions? - Stack Overflow](https://stackoverflow.com/questions/58594343/how-to-handle-keys-and-credentials-when-deploying-to-google-cloud-functions)

This repo deploys a data pipeline on GCP using the following serverless services:
* Cloud Functions
* Cloud Pub/Sub
* Cloud Scheduler
* BigQuery
* Cloud Storage

The aim is to query data within BigQuery using Cloud Functions and exporting the results in JSON format to Cloud Storage in JSON:
![](https://i.ibb.co/F8KKrf0/workflow-diagram.png)

## Instructions
* GCP account setup:
	1. Create project
	2. Create service account and download credentials json
	3. Give service account permissions to project
	4. Give service account permission to dataset or table(s)

* [Install Google Cloud SDK](https://cloud.google.com/sdk/docs/install)

* Place `google-cloud-sdk` folder in home directory

* Use the install script to add Cloud SDK tools to your path
```bash
~/google-cloud-sdk/install.sh
```

* Set the environment variable `GOOGLE_APPLICATION_CREDENTIALS`  so that client library can automatically find the service account credentials [see link](https://cloud.google.com/docs/authentication/production#automatically).

* [Initialise Cloud SDK](https://cloud.google.com/sdk/docs/initializing)
```bash
# authorise cloud sdk to use service account instead of user account to access GCP
# --key-file="path/to/service-account-credentials.json"
gcloud auth activate-service-account \
	--key-file="/Users/ayubk/json-todd-credentials.json" \
	--project="json-todd"
```

* Enable following APIs in GCP project:
	1. [BigQuery](https://console.cloud.google.com/apis/library/bigquery.googleapis.com)
	2. [Cloud Storage](https://console.cloud.google.com/apis/library/storage-component.googleapis.com)
	3. [Cloud Functions](https://console.cloud.google.com/apis/library/cloudfunctions.googleapis.com)
	4. [Cloud Build](https://console.cloud.google.com/apis/library/cloudbuild.googleapis.com)
	5. [Cloud Scheduler](https://console.cloud.google.com/apis/library/cloudscheduler.googleapis.com)
	6. [App Engine](https://console.cloud.google.com/apis/library/appengine.googleapis.com)
	7. [Cloud Pub/Sub](https://console.cloud.google.com/apis/library/pubsub.googleapis.com)

```bash
gcloud services enable \
	--account="myproject@myproject.iam.gserviceaccount.com" \
	appengine.googleapis.com \
	bigquery.googleapis.com \
	cloudbuild.googleapis.com \
	cloudfunctions.googleapis.com \
	cloudscheduler.googleapis.com \
	pubsub.googleapis.com \
	serviceusage.googleapis.com \
	storage-component.googleapis.com
```

* Create `requirements.txt` e.g.
```
google-cloud-bigquery
google-cloud-storage
```

* Create sql query file e.g. `count_by_date_query.sql` or a folder containing sql queries e.g. `queries/query1.sql, queries/query2.sql` etc.

* Create `config.py` file e.g.
```python
config_vars = {
    "project_id": "[ENTER YOUR PROJECT ID HERE]",
    "output_dataset_id": "[ENTER OUTPUT DATASET HERE]",
    "output_table_name": "[ENTER OUTPUT TABLE NAME HERE]",
    "sql_file_path": "github_query.sql"
}
```

* Create your `main.py` file (ensure it’s main and not e.g. `etl.py` or `process.py` etc.)
```python
# enter functions
def main(data, context): # must add data, context as args
	...

if __name__ == "__main__":
	main("date", "context")
```

* Go to directory where `main.py` is and deploy the function on GCP (and create PubSub topic specifying that this function is triggered whenever a new message is published to this topic)
	* [Function Identity | Cloud Functions Documentation | Google Cloud](https://cloud.google.com/functions/docs/securing/function-identity#deploying_a_new_function_with_a_non-default_identity)
```bash
# DEPLOY FUNCTION
gcloud functions deploy test_function \
	--service-account "json-todd@appspot.gserviceaccount.com" \
	--project="json-todd" \
	--entry-point main \
	--runtime python37 \
	--trigger-resource test_topic \
	--trigger-event google.pubsub.topic.publish \
	--timeout 540s
```

```bash
# list pubsub topics
gcloud pubsub topics list

# list cloud functions
gcloud functions list
```

* Schedule function (by scheduling message publish to above PubSub topic)
```bash
# SCHEDULE FUNCTION
gcloud scheduler jobs create pubsub test_schedule \
	--schedule "* */10 * * *" \
	--topic test_topic \
	--message-body "This is a test job that runs every 10 minutes."
```

```bash
# list scheduler jobs
gcloud beta scheduler jobs list
```

* Test run the job in Cloud Scheduler and check Logging.

## Notes
* If interacting with GCS API - a bucket named `gcf-sources-****` will be created. Don’t delete that. [SO ref](https://stackoverflow.com/questions/64842623/unable-to-deploy-google-cloud-functions)
	* `gcf-sources-<PROJECT_NUMBER>-<REGION>`
	* `gcf-sources-83963145828-us-central-1`
* Cloud Build - [required to build code into container image used by Cloud Functions](https://cloud.google.com/functions/docs/deploying#deployment)
* App Engine Admin API - [required for running Cloud Scheduler](https://cloud.google.com/scheduler/docs/tut-pub-sub#before-you-begin)
* If `iam.serviceAccounts.actAs` permission not added to service account then 403 error occurs. Add `iam.serviceAccounts.actAs` to service via a custom role.

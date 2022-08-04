# Purpose

Deploy and run a serverless app that converts Word documents (uploaded to a GCS bucket) to PDF.

## How To
#### Build the go server application
```
go build -o server
```

#### Build a container and put it in the Container Registry of the GCP project.
```
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter
```

#### Deploy the newly built container to Cloud Run
```
gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-east1 \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --set-env-vars PDF_BUCKET=$GOOGLE_CLOUD_PROJECT-processed \
  --max-instances=3
```
Note: It's a good idea to give LibreOffice 2GB of RAM to work with, see the line with the `--memory` option.

#### Create a Pub/Sub notification to indicate a new file has been uploaded to the docs bucket (`{PROJECT_ID}-upload`).
```
gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload
```
Note: The notifications will be labeled with the topic `new-doc`.

#### Create a new service account to trigger the Cloud Run services:
```
gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"

gcloud run services add-iam-policy-binding pdf-converter \
  --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
  --role=roles/run.invoker \
  --region us-east1 \
  --platform managed
  
PROJECT_NUMBER=$(gcloud projects list \
 --format="value(PROJECT_NUMBER)" \
 --filter="$GOOGLE_CLOUD_PROJECT")

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountTokenCreator
```

### Testing the Cloud Run service

#### Save the URL of the service in an environment variable:
```
SERVICE_URL=$(gcloud run services describe pdf-converter \
  --platform managed \
  --region us-east1 \
  --format "value(status.url)")
```

#### Print the contents of the new env var to confirm the URL:
```
echo $SERVICE_URL
```

#### Make an anonymous GET request to your new service:
```
curl -X GET $SERVICE_URL
```
Note: An authorization error should be returned to the client: `Your client does not have permission to get URL`.

#### Try invoking the service as an authorized user instead:
```
curl -X GET -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```

### Triggering and testing the conversion of files uploaded to the GCS bucket

#### Create a Pub/Sub subscription so that the PDF converter will be run whenever a message is published to the topic `new-doc`:
```
gcloud pubsub subscriptions create pdf-conv-sub \
  --topic new-doc \
  --push-endpoint=$SERVICE_URL \
  --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

Finally, upload Word files to the `{GOOGLE_CLOUD_PROJECT}-upload` bucket and check their PDF version are added to the `{GOOGLE_CLOUD_PROJECT}-processed` bucket.

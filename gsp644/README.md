# Purpose 

Deploy and run a serverless app that converts files (uploaded to a GCS bucket) to PDF.

## How To
#### Download NPM dependencies required by the app
```
npm install
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
  --no-allow-unauthenticated \
  --max-instances=1
```

#### Create a GCS bucket for the uploaded docs:
```
gsutil mb gs://$GOOGLE_CLOUD_PROJECT-upload
```

#### Add another GCS bucket for the processed PDFs:
```
gsutil mb gs://$GOOGLE_CLOUD_PROJECT-processed
```

#### Set up a Pub/Sub notification when a new file is uploaded to a bucket:
```
gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload
```

#### Create a service account which Pub/Sub will use to trigger the Cloud Run services:
```
gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
```

#### Give the new service account permission to invoke the PDF converter service:

```
gcloud beta run services add-iam-policy-binding pdf-converter --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --platform managed --region us-east1
```

#### Find your project number by running this command:
```
gcloud projects list
```

#### Create a `PROJECT_NUMBER={project_number}` ENV var with the project number obtained in the previous step
```
PROJECT_NUMBER=[project number]
```

#### Finally, create a Pub/Sub subscription so that the PDF converter can run whenever a message is published on the topic `new-doc`:
```
gcloud beta pubsub subscriptions create pdf-conv-sub --topic new-doc --push-endpoint=$SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

#### See if the Cloud Run service is triggered by uploading files to the `gs://$GOOGLE_CLOUD_PROJECT-upload` GCS bucket.
```
gsutil -m cp files/* gs://$GOOGLE_CLOUD_PROJECT-upload
```

And check the GCS bucket using the Google Cloud console, opening the **Navigation menu** and selecting the **Cloud Storage** option. Open the `-upload` bucket and click on the **Refresh** button a couple of times to see how files are deleted, one by one, as they are converted to PDF.


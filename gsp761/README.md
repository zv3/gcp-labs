# How to deploy and run the REST API app.

#### Build the server's app binary
```
go build -o server
```

#### Build a container and put it in the Container Registry of the GCP project.
```
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1
```

#### Deploy the newly built container to Cloud Run
```
gcloud run deploy rest-api \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1 \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --max-instances=2
```

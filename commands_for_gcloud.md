# RAG Scale Test: Execution \& Deployment Guide

This guide provides the necessary commands to set up, run locally, and deploy the Enterprise Agentic RAG application to Google Cloud Platform.

\---

## 🛠️ Useful Helper Commands

Use these commands to quickly find project details or check your status:

```powershell
# Get your Project Number (needed for Service Accounts)
gcloud projects describe --format="value(projectNumber)"

# See which account is currently logged in
gcloud auth list

# See all current configuration (Project ID, Region, Account)
gcloud config list

# List all enabled APIs in this project
gcloud services list --enabled

# List all VPC connectors in a region
gcloud compute networks vpc-access connectors list --region us-central1
```

\---

## 1\. Google Cloud Initial Setup (Terminal)

Before running any cloud-related commands, ensure you have the `gcloud` CLI installed and authenticated.

### Authentication \& Project Configuration

```powershell
# Login to Google Cloud
gcloud auth login

# Login for Application Default Credentials (needed for local python scripts)
gcloud auth application-default login

# Set the active project
gcloud config set project dmtxpress

# Enable required Google Cloud Services
gcloud services enable \\
    artifactregistry.googleapis.com \\
    run.googleapis.com \\
    cloudbuild.googleapis.com \\
    sqladmin.googleapis.com \\
    documentai.googleapis.com \\
    compute.googleapis.com \\
    discoveryengine.googleapis.com

# Create GCS Buckets (if they don't exist)
gcloud storage buckets create gs://dmtxpress-rag-raw --location=us-central1
gcloud storage buckets create gs://dmtxpress-rag-processed --location=us-central1
```

### IAM Permissions (Roles)

Run these to ensure your account has the necessary permissions:

```powershell
gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
  --member="user:adedamolaogunleye1@gmail.com" \\
    --role="roles/run.admin"

gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
  --member="user:adedamolaogunleye1@gmail.com" \\
    --role="roles/artifactregistry.admin"

gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
  --member="user:adedamolaogunleye1@gmail.com" \\
    --role="roles/storage.admin"

gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
  --member="user:adedamolaogunleye1@gmail.com" \\
    --role="roles/cloudsql.client"

gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
  --member="user:adedamolaogunleye1@gmail.com" \\
    --role="roles/aiplatform.user"

gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
  --member="user:adedamolaogunleye1@gmail.com" \\
    --role="roles/documentai.apiUser"

gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
  --member="user:adedamolaogunleye1@gmail.com" \\
    --role="roles/discoveryengine.editor"

# Grant Document AI access to the Cloud Run Service Account
gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
    --member="serviceAccount:647040633947-compute@developer.gserviceaccount.com" \\
    --role="roles/documentai.apiUser"

# Grant VPC access to the Cloud Run Service Agent
gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
    --member="serviceAccount:service-647040633947@serverless-robot-prod.iam.gserviceaccount.com" \\
    --role="roles/vpcaccess.user"

# Grant permission to the Cloud Run Service Account (Production)
gcloud projects add-iam-policy-binding enterprise-rag-510104 \\
    --member="serviceAccount:647040633947-compute@developer.gserviceaccount.com" \\
    --role="roles/discoveryengine.editor"
```

\---

## 2\. Local Environment Setup

### Virtual Environment \& Dependencies

```powershell
# Create a virtual environment
python -m venv venv

# Activate the virtual environment
.\\venv\\Scripts\\activate

# Install dependencies
pip install -r requirements.txt
```

### Environment Variables (.env)

Create a `.env` file in the root directory and paste the following:

```env
PROJECT\_ID="dmtxpress"
LOCATION="us-central1"
GCP\_DOC\_AI\_LOCATION="us"
GCP\_DOC\_AI\_PROCESSOR\_ID=""
GCP\_RAW\_BUCKET="dmtxpress-rag-raw"
GCP\_PROCESSED\_BUCKET="dmtxpress-rag-processed"
VPC\_CONNECTOR="rag-vps"

QDRANT\_API\_KEY=""
QDRANT\_CLUSTER\_ENDPOINT=""

GROQ\_API\_KEY=""

LOGFIRE\_TOKEN=""

LANGSMITH\_TRACING=true
LANGSMITH\_ENDPOINT=
LANGSMITH\_API\_KEY=
LANGSMITH\_PROJECT=
```

\---

## 3\. Data Ingestion

The ingestion pipeline is now **Universal**. It scans the `DATA/` directory, automatically identifies "true" and "noisy" subfolders, parses PDF/HTML/TXT, and syncs everything to GCP.

### Universal Ingestion (Recommended)

This command will process all subfolders in `DATA/` and map them to the correct buckets/tags.

```powershell
# Ingest everything in the DATA folder
python -m app.ingestion.processor DATA --wipe
```

### Manual Ingestion (Specific Folder)

```powershell
# Process a specific folder as a specific source type
python -m app.ingestion.processor DATA/true\_data true
```

> \[!TIP]
> The new pipeline now supports \*\*HTML\*\* files! Just drop your `.html` files into the `DATA/` subfolders.

\---

## 4\. Running Locally

### Start the FastAPI Backend

```powershell
uvicorn app.main:app --reload --port 8000
```

### Start the Streamlit UI

```powershell
streamlit run ui/app.py
```

\---

## 5\. Build and Push Image (No Local Docker Required)

Use Google Cloud Build to build the container image in the cloud and push it directly to Artifact Registry.

### Create Repository

```powershell
# Create a Docker repository in Artifact Registry
gcloud artifacts repositories create rag-repo \\
    --repository-format=docker \\
    --location=us-central1 \\
    --description="Docker repository for RAG API"
```

### Build and Push using Cloud Build

```powershell
# Submit a build to Google Cloud Build (this builds the image in the cloud and pushes it)
gcloud builds submit --tag us-central1-docker.pkg.dev/dmtxpress/rag-repo/rag-api:v1 .
```

### 6\. Create a vpc connector:

Note that underscores (\_) are not allowed in VPC connector names. You must use a hyphen (-) instead.

```
gcloud compute networks vpc-access connectors create rag-vps \\
    --region us-central1 \\
    --network default \\
    --range 10.8.0.0/28

```

\---

## 7\. Cloud Run Deployment

Deploy the containerized app to Google Cloud Run.

```powershell

gcloud run deploy rag-api \\
  --image us-central1-docker.pkg.dev/dmtxpress/rag-repo/rag-api:v1 \\
  --region us-central1 \\
  --platform managed \\
  --allow-unauthenticated \\
  --memory 2Gi \\
  --timeout=300 \\
  --vpc-connector rag-vps \\
  --set-env-vars "PROJECT\_ID=dmtxpress" \\
  --set-env-vars "LOCATION=us-central1" \\
  --set-env-vars "GCP\_DOC\_AI\_PROCESSOR\_ID=" \\
  --set-env-vars "GCP\_RAW\_BUCKET=dmtxpress-rag-raw" \\
  --set-env-vars "GCP\_PROCESSED\_BUCKET=dmtxpress-rag-processed" \\
  --set-env-vars "QDRANT\_API\_KEY=

&#x20; --set-env-vars "QDRANT\_CLUSTER\_ENDPOINT=
  --set-env-vars "GROQ\_API\_KEY" \\
  --set-env-vars "LOGFIRE\_TOKEN=F" \\
  --set-env-vars "LANGSMITH\_TRACING=true" \\
  --set-env-vars "LANGSMITH\_PROJECT=entreprise\_rag" \\
  --set-env-vars "LANGSMITH\_API\_KEY=" \\
  --set-env-vars "LANGSMITH\_ENDPOINT=https://api.smith.langchain.com"


```

\---

## 🚀 Scalable Architecture Setup

Follow these steps to upgrade your project to the scalable enterprise architecture using Terraform, Microservices, and Eventarc.

### 1\. Enable Extra Google Services

To support Redis, Eventarc, and Terraform management, enable these APIs:

```powershell
gcloud services enable \\
    redis.googleapis.com \\
    eventarc.googleapis.com \\
    pubsub.googleapis.com \\
    iam.googleapis.com \\
    cloudresourcemanager.googleapis.com
```

### 2\. Grant Extra Permissions (Scalability Roles)

These roles allow your account to create the advanced infrastructure and allow Eventarc to communicate.

```powershell
# Admin roles for infrastructure creation
gcloud projects add-iam-policy-binding \\
    --member="user:djadhwani20@gmail.com" \\
    --role="roles/eventarc.admin"

gcloud projects add-iam-policy-binding \\
    --member="user:djadhwani20@gmail.com" \\
    --role="roles/redis.admin"

gcloud projects add-iam-policy-binding \\
    --member="user:djadhwani20@gmail.com" \\
    --role="roles/iam.serviceAccountAdmin"

gcloud projects add-iam-policy-binding \\
    --member="user:djadhwani20@gmail.com" \\
    --role="roles/resourcemanager.projectIamAdmin"

gcloud projects add-iam-policy-binding \\
    --member="user:djadhwani20@gmail.com" \\
    --role="roles/run.admin"

gcloud projects add-iam-policy-binding \\
    --member="user:djadhwani20@gmail.com" \\
    --role="roles/compute.networkAdmin"

gcloud projects add-iam-policy-binding \\
    --member="user:djadhwani20@gmail.com" \\
    --role="roles/vpcaccess.admin"

# Allow Pub/Sub to publish GCS events (required for Eventarc)
gcloud projects add-iam-policy-binding \\
    --member="serviceAccount:service-$(gcloud projects describe --format='value(projectNumber)')@gs-project-accounts.iam.gserviceaccount.com" \\
    --role="roles/pubsub.publisher"
```

### 3\. Build Microservices (UI, Backend, Ingestion)

Instead of manual builds, use the new `cloudbuild.yaml` to build all three images at once in the cloud.

```powershell
# Submit a multi-service build
gcloud builds submit --config cloudbuild.yaml .
```

### 4\. Deploy Infrastructure with Terraform

Navigate to the `terraform/` folder and run these commands to spin up the Database, Redis, VPC, and Cloud Run services automatically.

```powershell
# Navigate to the folder (Manual Step)
cd terraform

# Initialize Terraform
terraform init

# Preview the changes
terraform plan

# Build Artifact repo first for smooth process

terraform apply -target=google\_artifact\_registry\_repository.repo

# Apply the changes (type 'yes' when prompted)
terraform apply

# destroy once everything is done

terraform destroy

```

### 5\. Final Verification Logic

Once deployed, verify your enterprise-grade RAG:

1. **The Ingestion Test**: Upload a PDF to your `dmtxpress-rag-raw` bucket. Check the Cloud Run logs for the `ingestion` service—you should see it wake up automatically!
2. **The Memory Test**: Ask a question in the UI. Refresh the page. The agent should remember your name/context because it's now in **Postgres**.
3. **The Speed Test**: Ask the same question twice. The second response should be instant (**< 100ms**) because of the **Redis Semantic Cache**.

\---

\---


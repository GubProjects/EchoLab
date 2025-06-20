\🚀 GitHub Actions CI/CD to GKE using Workload Identity Federation
This repository provides a secure and automated CI/CD pipeline to deploy applications to a Google Kubernetes Engine (GKE) cluster using GitHub Actions and Workload Identity Federation (WIF). This approach eliminates the need for long-lived credentials by leveraging short-lived OIDC tokens.

📋 Overview
This guide walks you through setting up a GitHub Actions workflow to deploy an application (e.g., an echoserver) to a GKE cluster. It uses Workload Identity Federation to securely authenticate GitHub Actions with Google Cloud Platform (GCP) without storing sensitive service account keys in your repository.
By the end of this guide, you will:

Create a Workload Identity Pool and Provider in GCP.
Set up a GCP service account with appropriate permissions.
Configure GitHub Actions to authenticate and deploy to GKE.
Deploy Kubernetes manifests (deployment.yaml and service.yaml) to your GKE cluster.
Understand how to clean up resources if needed.


📁 Project Structure
.
├── .github
│   └── workflows
│       └── deploy-gke.yaml      # GitHub Actions workflow for CI/CD
├── k8s
│   ├── deployment.yaml          # Kubernetes deployment manifest
│   ├── service.yaml             # Kubernetes service manifest
└── README.md                    # This file


🛠️ Prerequisites
Before starting, ensure you have the following:

Google Cloud Project:

A GCP project with billing enabled.
The GKE API enabled (container.googleapis.com).
A GKE cluster created in your project (e.g., gke-central1-cluster in us-central1-a).


Google Cloud SDK (gcloud):

Installed locally or in a Cloud Shell environment.
Authenticated with a user account that has Owner or Editor permissions for the GCP project.


GitHub Repository:

A GitHub repository (e.g., your-username/echoserver-lab) with admin access.
The repository should contain the Kubernetes manifests in the k8s/ directory.


kubectl:

Installed locally for testing and verification.
Configured to interact with your GKE cluster (optional for local testing).


Basic Knowledge:

Familiarity with GitHub Actions, Kubernetes, and GCP concepts like service accounts and IAM.




🔧 Placeholder Reference
The following placeholders are used throughout this guide. Replace them with your specific values:



Placeholder
Description



<PROJECT_ID>
Your GCP project ID (e.g., my-gcp-project-123).


<PROJECT_NUMBER>
Your GCP project number (found in the GCP Console under "Project Info").


<GITHUB_REPO>
Your GitHub repository (e.g., your-username/echoserver-lab).


<WORKLOAD_POOL_NAME>
Name for the Workload Identity Pool (e.g., github-wif-pool).


<WIF_PROVIDER_NAME>
Name for the Workload Identity Provider (e.g., github-oidc-provider).


<SERVICE_ACCOUNT_EMAIL>
GCP service account email (e.g., gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com).


How to Find These Values

Project ID: In the GCP Console, go to Dashboard → Project Info.
Project Number: Same as above, under Project Info.
GitHub Repo: The format is <owner>/<repo-name> (e.g., derrickSh43/echoserver-lab).
Workload Pool and Provider Names: Choose unique names (e.g., github-wif-pool and github-oidc-provider).
Service Account Email: Generated in Step 2 below.


🏗️ Step-by-Step Setup Guide
Step 1: Set Up Your GKE Cluster

Create a GKE Cluster (if not already created):

In the GCP Console, navigate to Kubernetes Engine → Clusters → Create Cluster.
Choose a Standard Cluster or Autopilot Cluster based on your needs.
Set the cluster name (e.g., gke-central1-cluster) and location (e.g., us-central1-a).
Enable Workload Identity on the cluster (under Security settings during creation).
Deploy the cluster and wait for it to be ready.


Verify Cluster Access:

Install gcloud and kubectl locally if not already installed.
Run the following to authenticate and configure kubectl:gcloud auth login
gcloud container clusters get-credentials gke-central1-cluster --zone us-central1-a --project <PROJECT_ID>
kubectl get nodes


Ensure you see the cluster nodes in the output.



Step 2: Create Workload Identity Pool and Provider

Create a Workload Identity Pool:

Run the following command to create a Workload Identity Pool in GCP:gcloud iam workload-identity-pools create <WORKLOAD_POOL_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --display-name="GitHub WIF Pool"


Replace <WORKLOAD_POOL_NAME> with a unique name (e.g., github-wif-pool).
Replace <PROJECT_ID> with your GCP project ID.


Create a Workload Identity Provider:

Create an OIDC provider linked to GitHub Actions:gcloud iam workload-identity-pools providers create-oidc <WIF_PROVIDER_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>" \
  --display-name="GitHub OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-condition="attribute.repository == '<GITHUB_REPO>'"


Replace <WIF_PROVIDER_NAME> with a unique name (e.g., github-oidc-provider).
Replace <GITHUB_REPO> with your repository (e.g., your-username/echoserver-lab).


Verify Creation:

List the Workload Identity Pools to confirm:gcloud iam workload-identity-pools list --project="<PROJECT_ID>" --location="global"


List the providers in the pool:gcloud iam workload-identity-pools providers list \
  --project="<PROJECT_ID>" \
  --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>"





Step 3: Create and Bind GCP Service Account

Create a Service Account:

Create a service account for GitHub Actions to use:gcloud iam service-accounts create gke-lab-sa \
  --project="<PROJECT_ID>" \
  --display-name="GKE Deploy Service Account"


This creates a service account named gke-lab-sa. Note the email: gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com.


Grant Permissions to the Service Account:

Assign the necessary roles to the service account for GKE operations:gcloud projects add-iam-policyව





System: Thank you for sharing your repository details! I'll help create a comprehensive, step-by-step README for your GitHub Actions CI/CD to GKE using Workload Identity Federation. Below is a detailed guide that expands on your existing README, covering every step explicitly with no assumptions left to the imagination.

# 🚀 GitHub Actions CI/CD to GKE using Workload Identity Federation

This repository provides a secure and automated CI/CD pipeline to deploy applications to a Google Kubernetes Engine (GKE) cluster using GitHub Actions and Workload Identity Federation (WIF). This approach eliminates the need for long-lived credentials by leveraging short-lived OIDC tokens.

📋 Overview
This guide walks you through setting up a GitHub Actions workflow to deploy an application (e.g., an echoserver) to a GKE cluster. It uses Workload Identity Federation to securely authenticate GitHub Actions with Google Cloud Platform (GCP) without storing sensitive service account keys in your repository.
By the end of this guide, you will:

Create a Workload Identity Pool and Provider in GCP.
Set up a GCP service account with appropriate permissions.
Configure GitHub Actions to authenticate and deploy to GKE.
Deploy Kubernetes manifests (deployment.yaml and service.yaml) to your GKE cluster.
Understand how to clean up resources if needed.


📁 Project Structure
.
├── .github
│   └── workflows
│       └── deploy-gke.yaml      # GitHub Actions workflow for CI/CD
├── k8s
│   ├── deployment.yaml          # Kubernetes deployment manifest
│   ├── service.yaml             # Kubernetes service manifest
└── README.md                    # This file


🛠️ Prerequisites
Before starting, ensure you have the following:

Google Cloud Project:

A GCP project with billing enabled.
The GKE API enabled (container.googleapis.com).
A GKE cluster created in your project (e.g., gke-central1-cluster in us-central1-a).


Google Cloud SDK (gcloud):

Installed locally or in a Cloud Shell environment.
Authenticated with a user account that has Owner or Editor permissions for the GCP project.


GitHub Repository:

A GitHub repository (e.g., your-username/echoserver-lab) with admin access.
The repository should contain the Kubernetes manifests in the k8s/ directory.


kubectl:

Installed locally for testing and verification.
Configured to interact with your GKE cluster (optional for local testing).


Basic Knowledge:

Familiarity with GitHub Actions, Kubernetes, and GCP concepts like service accounts and IAM.




🔧 Placeholder Reference
The following placeholders are used throughout this guide. Replace them with your specific values:



Placeholder
Description



<PROJECT_ID>
Your GCP project ID (e.g., my-gcp-project-123).


<PROJECT_NUMBER>
Your GCP project number (found in the GCP Console under "Project Info").


<GITHUB_REPO>
Your GitHub repository (e.g., your-username/echoserver-lab).


<WORKLOAD_POOL_NAME>
Name for the Workload Identity Pool (e.g., github-wif-pool).


<WIF_PROVIDER_NAME>
Name for the Workload Identity Provider (e.g., github-oidc-provider).


<SERVICE_ACCOUNT_EMAIL>
GCP service account email (e.g., gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com).


How to Find These Values

Project ID: In the GCP Console, go to Dashboard → Project Info.
Project Number: Same as above, under Project Info.
GitHub Repo: The format is <owner>/<repo-name> (e.g., derrickSh43/echoserver-lab).
Workload Pool and Provider Names: Choose unique names (e.g., github-wif-pool and github-oidc-provider).
Service Account Email: Generated in Step 3 below.


🏗️ Step-by-Step Setup Guide
Step 1: Set Up Your GKE Cluster

Create a GKE Cluster (if not already created):

In the GCP Console, navigate to Kubernetes Engine → Clusters → Create Cluster.
Choose a Standard Cluster or Autopilot Cluster based on your needs.
Set the cluster name (e.g., gke-central1-cluster) and location (e.g., us-central1-a).
Enable Workload Identity on the cluster (under Security settings during creation).
Deploy the cluster and wait for it to be ready (this may take 5-10 minutes).
Note the cluster name and zone for use in the GitHub Actions workflow.


Verify Cluster Access:

Install gcloud and kubectl locally if not already installed:
gcloud: Follow Google's installation guide.
kubectl: Install via gcloud components install kubectl or follow Kubernetes docs.


Authenticate gcloud:gcloud auth login


Configure kubectl to connect to your GKE cluster:gcloud container clusters get-credentials gke-central1-cluster --zone us-central1-a --project <PROJECT_ID>


Verify connectivity:kubectl get nodes


You should see a list of nodes in your GKE cluster. If not, ensure your cluster is running and your credentials are correct.



Step 2: Create Kubernetes Manifests

Create the k8s/ Directory:

In your repository, create a directory named k8s:mkdir k8s




Create deployment.yaml:

This defines an echoserver deployment with 2 replicas.
Save the following in k8s/deployment.yaml:apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
      - name: echoserver
        image: k8s.gcr.io/echoserver:1.4
        ports:
        - containerPort: 8080




Create service.yaml:

This exposes the echoserver deployment as a LoadBalancer service.
Save the following in k8s/service.yaml:apiVersion: v1
kind: Service
metadata:
  name: echoserver-service
  namespace: prod
spec:
  selector:
    app: echoserver
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer




Commit and Push Manifests:

Add the files to your repository:git add k8s/deployment.yaml k8s/service.yaml
git commit -m "Add Kubernetes manifests for echoserver"
git push origin main





Step 3: Create Workload Identity Pool and Provider

Create a Workload Identity Pool:

Run the following command to create a Workload Identity Pool in GCP:gcloud iam workload-identity-pools create <WORKLOAD_POOL_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --display-name="GitHub WIF Pool"


Replace <WORKLOAD_POOL_NAME> with a unique name (e.g., github-wif-pool).
Replace <PROJECT_ID> with your GCP project ID (e.g., my-gcp-project-123).


Create a Workload Identity Provider:

Create an OIDC provider linked to GitHub Actions:gcloud iam workload-identity-pools providers create-oidc <WIF_PROVIDER_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>" \
  --display-name="GitHub OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-condition="attribute.repository == '<GITHUB_REPO>'"


Replace <WIF_PROVIDER_NAME> with a unique name (e.g., github-oidc-provider).
Replace <GITHUB_REPO> with your repository (e.g., your-username/echoserver-lab).


Verify Creation:

List the Workload Identity Pools:gcloud iam workload-identity-pools list --project="<PROJECT_ID>" --location="global"


List the providers in the pool:gcloud iam workload-identity-pools providers list \
  --project="<PROJECT_ID>" \
  --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>"


Ensure your pool and provider appear in the output.



Step 4: Create and Bind GCP Service Account

Create a Service Account:

Create a service account for GitHub Actions:gcloud iam service-accounts create gke-lab-sa \
  --project="<PROJECT_ID>" \
  --display-name="GKE Deploy Service Account"


Note the service account email: gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com.


Grant Permissions to the Service Account:

Assign the following roles to allow GKE operations:gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/container.developer"
gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"




Bind the Service Account to the Workload Identity Pool:

Allow the GitHub repository to impersonate the service account:gcloud iam service-accounts add-iam-policy-binding gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com \
  --project="<PROJECT_ID>" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<WORKLOAD_POOL_NAME>/attribute.repository/<GITHUB_REPO>"





Step 5: Add GitHub Repository Secret

Navigate to Repository Settings:

Go to your GitHub repository in a browser.
Click Settings → Secrets and variables → Actions → New repository secret.


Add the GKE Project Secret:

Name: GKE_PROJECT
Value: <PROJECT_ID> (your GCP project ID, e.g., my-gcp-project-123).
Click Add secret.



Step 6: Configure GitHub Actions Workflow

Create the Workflow File:

The provided deploy-gke.yaml is already well-structured. Below is the complete workflow with placeholders replaced for clarity.
Save the following in .github/workflows/deploy-gke.yaml:name: Deploy Echoserver to GKE
on:
  push:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GKE_PROJECT }}
  GKE_CLUSTER: gke-central1-cluster
  GKE_ZONE: us-central1-a
  DEPLOYMENT_NAME: echoserver
  NAMESPACE: prod
  PROJECT_NUMBER: <PROJECT_NUMBER>
  WORKLOAD_POOL_NAME: <WORKLOAD_POOL_NAME>
  WIF_PROVIDER_NAME: <WIF_PROVIDER_NAME>

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<WORKLOAD_POOL_NAME>/providers/<WIF_PROVIDER_NAME>
          service_account: gke-lab-sa@${{ env.PROJECT_ID }}.iam.gserviceaccount.com

      - uses: google-github-actions/setup-gcloud@v2
        with:
          project_id: ${{ env.PROJECT_ID }}

      - uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: ${{ env.GKE_CLUSTER }}
          location: ${{ env.GKE_ZONE }}

      - name: Deploy to GKE
        run: |
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml
          kubectl rollout status deployment/${{ env.DEPLOYMENT_NAME }} -n ${{ env.NAMESPACE }}




Update Placeholders:

Replace <PROJECT_NUMBER>, <WORKLOAD_POOL_NAME>, and <WIF_PROVIDER_NAME> with your values from Steps 3 and 4.
Commit and push the workflow file:git add .github/workflows/deploy-gke.yaml
git commit -m "Add GitHub Actions workflow for GKE deployment"
git push origin main





Step 7: Test the Deployment

Trigger the Workflow:

Pushing to the main branch triggers the GitHub Actions workflow.
Monitor the workflow in the Actions tab of your GitHub repository.


Verify Deployment:

Once the workflow completes, get the external IP of the service:kubectl get svc -n prod


Look for the echoserver-service and note its EXTERNAL-IP.
Access the echoserver in a browser: http://<EXTERNAL-IP>.
You should see the echoserver response with request details.



Step 8: Monitor and Troubleshoot

Check Workflow Logs:

In the GitHub Actions tab, click the workflow run to view logs.
Look for errors in the auth, setup-gcloud, get-gke-credentials, or Deploy to GKE steps.


Common Issues and Fixes:

Authentication Failure: Ensure the <PROJECT_NUMBER>, <WORKLOAD_POOL_NAME>, <WIF_PROVIDER_NAME>, and <GITHUB_REPO> match exactly in all commands and the workflow file.
Permission Denied: Verify the service account has roles/container.developer and roles/iam.serviceAccountUser.
Cluster Not Found: Confirm the GKE_CLUSTER and GKE_ZONE match your cluster’s configuration.
Deployment Fails: Check the manifests for typos and ensure the prod namespace exists (kubectl create namespace prod).




🧹 Cleanup (Optional)
If you need to remove the resources created:

Delete the Workload Identity Provider:
gcloud iam workload-identity-pools providers delete <WIF_PROVIDER_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>"


Delete the Workload Identity Pool:
gcloud iam workload-identity-pools delete <WORKLOAD_POOL_NAME> \
  --project="<PROJECT_ID>" \
  --location="global"


Delete the Service Account:
gcloud iam service-accounts delete gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com \
  --project="<PROJECT_ID>"


Delete the GKE Cluster (if no longer needed):
gcloud container clusters delete gke-central1-cluster --zone us-central1-a --project <PROJECT_ID>


Remove GitHub Secret:

Go to Settings → Secrets and variables → Actions.
Delete the GKE_PROJECT secret.




📚 Additional Resources

Google Cloud Workload Identity Federation
GitHub Actions Documentation
Kubernetes Documentation
google-github-actions


⚠️ Notes

Ensure all commands are run in a terminal with gcloud authenticated.
The prod namespace must exist in your GKE cluster before deployment (kubectl create namespace prod).
The echoserver image (k8s.gcr.io/echoserver:1.4) is publicly


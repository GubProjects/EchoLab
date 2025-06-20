# 🚀 GitHub Actions CI/CD to GKE using Workload Identity Federation

This repository provides a secure and automated CI/CD pipeline to deploy applications to a Google Kubernetes Engine (GKE) cluster using GitHub Actions and Workload Identity Federation (WIF). This approach eliminates the need for long-lived credentials by leveraging short-lived OIDC tokens.

---

## 📋 Overview

This guide walks you through setting up a GitHub Actions workflow to deploy an application (e.g., an echoserver) to a GKE cluster. It uses Workload Identity Federation to securely authenticate GitHub Actions with Google Cloud Platform (GCP) without storing sensitive service account keys in your repository.

By the end of this guide, you will:

* Create a Workload Identity Pool and Provider in GCP.
* Set up a GCP service account with appropriate permissions.
* Configure GitHub Actions to authenticate and deploy to GKE.
* Deploy Kubernetes manifests to your GKE cluster.
* Understand how to clean up resources if needed.

---

## 📁 Project Structure

```
.
├── .github
│   └── workflows
│       └── deploy-gke.yaml      # GitHub Actions workflow
├── k8s
│   ├── deployment.yaml          # Kubernetes Deployment
│   ├── service.yaml             # Kubernetes Service
│   └── namespace.yaml           # Kubernetes Namespace (new)
└── README.md                    # This file
```

---

## 🛠️ Prerequisites

* GCP project with billing enabled and GKE API enabled.
* A GKE cluster (e.g., `gke-central1-cluster` in `us-central1-a`).
* `gcloud` and `kubectl` installed and configured.
* A GitHub repository with admin access.
* Basic knowledge of GitHub Actions, Kubernetes, and GCP IAM.

---

## 🔧 Placeholder Reference

| Placeholder               | Description                                             |
| ------------------------- | ------------------------------------------------------- |
| `<PROJECT_ID>`            | Your GCP project ID                                     |
| `<PROJECT_NUMBER>`        | Your GCP project number                                 |
| `<GITHUB_REPO>`           | Format: `username/repo-name`                            |
| `<WORKLOAD_POOL_NAME>`    | Workload Identity Pool name                             |
| `<WIF_PROVIDER_NAME>`     | Workload Identity Provider name                         |
| `<SERVICE_ACCOUNT_EMAIL>` | e.g., `gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com` |

---

## 🏋️ Step-by-Step Setup Guide

### Step 1: Set Up GKE Cluster

1. In GCP Console: Kubernetes Engine > Clusters > Create Cluster
2. Choose standard or autopilot cluster.
3. Enable Workload Identity under Security settings.
4. Note your cluster name and zone.

Verify access:

```bash
gcloud auth login
gcloud container clusters get-credentials gke-central1-cluster --zone us-central1-a --project <PROJECT_ID>
kubectl get nodes
```

---

### Step 2: Create Kubernetes Manifests

Create a `k8s/` directory and add the following files:

**`namespace.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

**`deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
  namespace: prod
  labels:
    app.kubernetes.io/name: echoserver
    app.kubernetes.io/version: "1.0"
  annotations:
    prometheus.io/scrape: "true"        # Enable Prometheus metrics scraping
    prometheus.io/port: "8080"          # Define the Prometheus target port
spec:
  replicas: 3                           # High availability with 3 replicas
  selector:
    matchLabels:
      app.kubernetes.io/name: echoserver
  template:
    metadata:
      labels:
        app.kubernetes.io/name: echoserver
        app.kubernetes.io/version: "1.0"
    spec:
      terminationGracePeriodSeconds: 10  # Allow app to shut down cleanly
      affinity:
        podAntiAffinity:                 # Spread pods across nodes when possible
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: echoserver
              topologyKey: kubernetes.io/hostname
      containers:
      - name: echoserver
        image: gcr.io/google_containers/echoserver:1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: PORT                    # Optional: Set explicit port environment variable
          value: "8080"
        resources:                      # Lightweight resource limits
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
        livenessProbe:                  # Restart if not responding
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:                # Traffic only routed to ready pods
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 2
      tolerations:
      - key: "cloud.google.com/gke-preemptible"  # Allow use of cheaper preemptible nodes
        operator: "Exists"
        effect: "NoSchedule"

```

**`service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echoserver
  namespace: prod
  labels:
    app.kubernetes.io/name: echoserver
    app.kubernetes.io/version: "1.0"
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  selector:
    app.kubernetes.io/name: echoserver

```

Apply and commit:

```bash
kubectl apply -f k8s/namespace.yaml
git add k8s/
git commit -m "Add echoserver manifests"
git push origin main
```

---

### Step 3: Create Workload Identity Pool and Provider

```bash
gcloud iam workload-identity-pools create <WORKLOAD_POOL_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --display-name="GitHub WIF Pool"

gcloud iam workload-identity-pools providers create-oidc <WIF_PROVIDER_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>" \
  --display-name="GitHub OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-condition="attribute.repository == '<GITHUB_REPO>'"
```

---

### Step 4: Create & Bind GCP Service Account

```bash
gcloud iam service-accounts create gke-lab-sa \
  --project="<PROJECT_ID>" \
  --display-name="GKE Deploy Service Account"

# Add roles
gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/container.developer"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# Bind GitHub repo to service account
gcloud iam service-accounts add-iam-policy-binding gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com \
  --project="<PROJECT_ID>" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<WORKLOAD_POOL_NAME>/attribute.repository/<GITHUB_REPO>"
```

---

### Step 5: Add GitHub Repository Secrets

In your GitHub repository:

* Go to `Settings > Secrets and variables > Actions`
* Add the following secrets:

  * `GKE_PROJECT`: `<PROJECT_ID>`
  * `GKE_CLUSTER`: `gke-central1-cluster`
  * `GKE_ZONE`: `us-central1-a`
  * `PROJECT_NUMBER`: `<PROJECT_NUMBER>`
  * `WORKLOAD_POOL_NAME`: `<WORKLOAD_POOL_NAME>`
  * `WIF_PROVIDER_NAME`: `<WIF_PROVIDER_NAME>`

---

### Step 6: Configure GitHub Actions Workflow

Create `.github/workflows/deploy-gke.yaml`:

```yaml
name: Deploy Echoserver to GKE
on:
  push:
    branches: [main]

env:
  PROJECT_ID: ${{ secrets.GKE_PROJECT }}
  GKE_CLUSTER: ${{ secrets.GKE_CLUSTER }}
  GKE_ZONE: ${{ secrets.GKE_ZONE }}
  DEPLOYMENT_NAME: echoserver
  NAMESPACE: prod
  PROJECT_NUMBER: ${{ secrets.PROJECT_NUMBER }}
  WORKLOAD_POOL_NAME: ${{ secrets.WORKLOAD_POOL_NAME }}
  WIF_PROVIDER_NAME: ${{ secrets.WIF_PROVIDER_NAME }}

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
        workload_identity_provider: "projects/${{ env.PROJECT_NUMBER }}/locations/global/workloadIdentityPools/${{ env.WORKLOAD_POOL_NAME }}/providers/${{ env.WIF_PROVIDER_NAME }}"
        service_account: "gke-lab-sa@${{ env.PROJECT_ID }}.iam.gserviceaccount.com"

    - uses: google-github-actions/setup-gcloud@v2
      with:
        project_id: ${{ env.PROJECT_ID }}

    - uses: google-github-actions/get-gke-credentials@v2
      with:
        cluster_name: ${{ env.GKE_CLUSTER }}
        location: ${{ env.GKE_ZONE }}

    - name: Deploy to GKE
      run: |
        kubectl apply -f k8s/namespace.yaml
        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml
        kubectl rollout status deployment/${{ env.DEPLOYMENT_NAME }} -n ${{ env.NAMESPACE }}
```

Commit and push:

```bash
git add .github/workflows/deploy-gke.yaml
git commit -m "Add deploy workflow"
git push origin main
```

---

### Step 7: Verify the Deployment

```bash
kubectl get pods -n prod
kubectl get svc -n prod
```

Visit `http://<EXTERNAL-IP>` to test.

---

## 🚼 Cleanup (Optional)

```bash
gcloud container clusters delete gke-central1-cluster --zone us-central1-a --project <PROJECT_ID>
gcloud iam service-accounts delete gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com --project <PROJECT_ID>
gcloud iam workload-identity-pools providers delete <WIF_PROVIDER_NAME> --project <PROJECT_ID> --location=global --workload-identity-pool <WORKLOAD_POOL_NAME>
gcloud iam workload-identity-pools delete <WORKLOAD_POOL_NAME> --project <PROJECT_ID> --location=global
```

Delete GitHub secrets if no longer needed.

---

## 📙 References

* [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
* [GitHub Actions for GCP](https://github.com/google-github-actions)
* [GKE Documentation](https://cloud.google.com/kubernetes-engine)
* [Echoserver](https://github.com/kubernetes-retired/contrib/tree/master/echoserver)

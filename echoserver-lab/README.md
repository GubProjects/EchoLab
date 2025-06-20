# 🚀 GitHub Actions CI/CD to GKE using Workload Identity Federation

Securely deploy to a GKE cluster using GitHub Actions with **Workload Identity Federation (WIF)** — no long-lived credentials needed.

---

## 📁 Project Structure

```
.
├── .github
│   └── workflows
│       └── deploy-gke.yaml
├── k8s
│   ├── deployment.yaml
│   ├── service.yaml
```

---

## 🔧 Placeholder Reference

| Placeholder               | Description                                                      |
|---------------------------|------------------------------------------------------------------|
| `<PROJECT_ID>`            | Your GCP project ID                                              |
| `<PROJECT_NUMBER>`        | Your GCP project number                                          |
| `<GITHUB_REPO>`           | GitHub repo (e.g., `derrickSh43/echoserver-lab`)                |
| `<WORKLOAD_POOL_NAME>`    | Workload Identity Pool name                                     |
| `<WIF_PROVIDER_NAME>`     | Workload Identity Provider name                                 |
| `<SERVICE_ACCOUNT_EMAIL>` | GCP deploy service account email                                |

---

## 🛠️ Step 1: Create WIF Pool & Provider

```bash
gcloud iam workload-identity-pools create <WORKLOAD_POOL_NAME> \
  --project="<PROJECT_ID>" --location="global" --display-name="GitHub WIF Pool"

gcloud iam workload-identity-pools providers create-oidc <WIF_PROVIDER_NAME> \
  --project="<PROJECT_ID>" --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>" \
  --display-name="GitHub OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-condition="attribute.repository == '<GITHUB_REPO>'"
```

---

## 🔐 Step 2: Create & Bind GCP Service Account

```bash
gcloud iam service-accounts create gke-lab-sa \
  --project="<PROJECT_ID>" --display-name="GKE Deploy Service Account"

gcloud iam service-accounts add-iam-policy-binding <SERVICE_ACCOUNT_EMAIL> \
  --project="<PROJECT_ID>" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<WORKLOAD_POOL_NAME>/attribute.repository/<GITHUB_REPO>"
```

---

## 🧾 Step 3: Add GitHub Repo Secret

- Go to: `Settings → Secrets and variables → Actions`
- Add secret: `GKE_PROJECT` = `<PROJECT_ID>`

---

## ⚙️ Step 4: GitHub Actions Workflow

```yaml
name: Deploy Echoserver to GKE
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
```

---

## 🧹 Cleanup Commands (Optional)

```bash
gcloud iam workload-identity-pools providers delete <WIF_PROVIDER_NAME> \
  --project="<PROJECT_ID>" --location="global" --workload-identity-pool="<WORKLOAD_POOL_NAME>"

gcloud iam workload-identity-pools delete <WORKLOAD_POOL_NAME> \
  --project="<PROJECT_ID>" --location="global"
```

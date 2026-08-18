# GKE_CLUSTER_CREATION_IN_WORKFLOW
# GitHub Actions → GCP Workload Identity Federation Setup

One-time IAM setup so GitHub Actions can authenticate to GCP **without a stored JSON key**, then create/manage your GKE cluster. Run these in Cloud Shell or local `gcloud` CLI (not inside the workflow).

---

### 1. Set project variables
```powershell
$PROJECT_ID="gke-cluster-creation-workflow"
$PROJECT_NUM=(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
$REPO="naveenkumaryeti/GKE_CLUSTER_CREATION_IN_WORKFLOW"
```
Reused by every command below. `$PROJECT_NUM` is needed for the trust binding in step 7.

### 2. Enable required APIs
```powershell
gcloud services enable iamcredentials.googleapis.com sts.googleapis.com container.googleapis.com --project=$PROJECT_ID
```
`iamcredentials` + `sts` power WIF token exchange; `container` powers GKE.

### 3. Create Workload Identity Pool
```powershell
gcloud iam workload-identity-pools create "github-pool" --project=$PROJECT_ID --location="global" --display-name="GitHub Pool"
```
A pool is a container for external identities (GitHub) that GCP will trust.

### 4. Create the WIF Provider (OIDC, GitHub)
```bash
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project="$PROJECT_ID" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository=='$REPO'" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```
Tells GCP to trust GitHub's OIDC tokens, restricted to your specific repo only.

### 5. Create a dedicated service account
```powershell
gcloud iam service-accounts create github-deployer --project=$PROJECT_ID --display-name="GitHub Actions Deployer"
```
This SA is what the workflow effectively "becomes" — least-privilege identity, not your personal account.

### 6. Grant GKE Admin role to the service account
```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:github-deployer@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/container.admin"
```
Permission to create/manage GKE clusters.

### 7. Allow the WIF pool to impersonate that service account
```bash
gcloud iam service-accounts add-iam-policy-binding "github-deployer@$PROJECT_ID.iam.gserviceaccount.com" \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUM/locations/global/workloadIdentityPools/github-pool/attribute.repository/$REPO"
```
Links GitHub's identity (from step 4) to the service account (from step 5) — this is the actual trust bridge.

### 8. Get the WIF_PROVIDER value
```bash
gcloud iam workload-identity-pools providers describe "github-provider" \
  --project="$PROJECT_ID" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --format="value(name)"
```
Copy the full output string — this is your `WIF_PROVIDER` secret value.

### 9. Add secrets to GitHub repo
Repo → **Settings → Secrets and variables → Actions → New repository secret**:
| Secret | Value |
|---|---|
| `WIF_PROVIDER` | output from step 8 |
| `WIF_SERVICE_ACCOUNT` | `github-deployer@gke-cluster-creation-workflow.iam.gserviceaccount.com` |

---

Once these secrets exist, `gke-create-cluster.yml` can authenticate keylessly and proceed to create/verify/print the GKE cluster.

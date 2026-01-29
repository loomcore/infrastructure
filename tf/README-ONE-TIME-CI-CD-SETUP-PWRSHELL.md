# One-time CI/CD Setup Guide (PowerShell Script)

This guide outlines the **one-time manual steps** required to bootstrap your Google Cloud Platform (GCP) organization and prepare it for automated Terraform deployments via GitHub Actions.

**Prerequisites:**
1.  You must have the **Google Cloud SDK** (`gcloud`) installed and initialized.
2.  Run these commands in **PowerShell**.
3.  Your account must have the following roles (at the org, folder, or project level as needed):
    - **Billing Account Administrator** — to link billing to new projects.
    - **Organization Administrator** (or equivalent) — to create projects and manage org-level settings.
    - **Organization Policy Administrator** (`roles/orgpolicy.policyAdmin`) on the projects (or a parent folder/org) — to set the `iam.allowedPolicyMemberDomains` policy so Cloud Run can allow public (`allUsers`) access.

### Installing Google Cloud SDK

If you don't have `gcloud` installed, you can install it using one of these methods:

**Option 1: Using PowerShell (Recommended for Windows)**
```powershell
# Download and run the Google Cloud SDK installer
(New-Object Net.WebClient).DownloadFile("https://dl.google.com/dl/cloudsdk/channels/rapid/GoogleCloudSDKInstaller.exe", "$env:Temp\GoogleCloudSDKInstaller.exe")
& $env:Temp\GoogleCloudSDKInstaller.exe
```

**Option 2: Using Chocolatey**
```powershell
choco install gcloudsdk
```

**Option 3: Manual Installation**
Download the installer from [Google Cloud SDK Downloads](https://cloud.google.com/sdk/docs/install) and follow the installation wizard.

After installation, initialize `gcloud`:
```powershell
gcloud init
```

This will prompt you to log in, select a project, and set default configuration values.

---

## Phase 0: Configuration Variables

Copy and paste this block into your PowerShell window to set up the variables we will use.
**IMPORTANT:** Replace the placeholder values with your actual IDs before running!

```powershell
# === CONFIGURATION ===
$ORG_ID = "123456789012"                    # Run 'gcloud organizations list' to find this (format: numeric string)
$FOLDER_ID = "987654321098"                 # The folder ID where projects will be created (format: numeric string)
                                            # Run 'gcloud resource-manager folders list --organization=$ORG_ID' to find this
$BILLING_ACCOUNT_ID = "01ABCD-123456-789ABC" # Run 'gcloud beta billing accounts list' to find this (format: alphanumeric with dashes)
$DOMAIN = "mydomain.com"                    # Your organization's domain name
$GITHUB_REPO = "my-org/infrastructure"      # Format: ORG/REPO (your GitHub organization and infra repo name)
$GITHUB_ORG = $GITHUB_REPO.Split('/')[0]    # Derived from GITHUB_REPO (e.g. "my-org"); used in WIF attribute conditions
$PROJECT_NAME = "project-name"

# Project IDs (must be globally unique, lowercase, alphanumeric and hyphens only)
$SHARED_PROJECT_ID = "project-name-shared"
$DEV_PROJECT_ID = "project-name-dev"
$PROD_PROJECT_ID = "project-name-prod"

# Resources
$TF_STATE_BUCKET = "org-name-terraform-state"   # Globally unique bucket name for Terraform state (lowercase, alphanumeric and hyphens)
$WIF_POOL_NAME = "github-pool"                  # Workload Identity Pool name
$WIF_PROVIDER_NAME = "github-provider"          # Workload Identity Provider name
$SA_DEV = "github-actions-dev"                  # Service account name for dev environment
$SA_PROD = "github-actions-prod"                # Service account name for prod environment
$REGION = "us-central1"                         # Or your preferred region (e.g., us-east1, europe-west1)
```

---

## Phase 1: Create Projects & Link Billing

Project creation is asynchronous in GCP. A short delay before linking billing (or running Phase 2) avoids "project not found" or similar errors.

```powershell
# Create Projects
gcloud projects create $SHARED_PROJECT_ID --name="Project Name Shared" --folder=$FOLDER_ID
gcloud projects create $DEV_PROJECT_ID --name="Project Name Dev" --folder=$FOLDER_ID
gcloud projects create $PROD_PROJECT_ID --name="Project Name Prod" --folder=$FOLDER_ID

# Wait for project creation to propagate before using projects
Start-Sleep -Seconds 10

# Link Billing
gcloud beta billing projects link $SHARED_PROJECT_ID --billing-account=$BILLING_ACCOUNT_ID
gcloud beta billing projects link $DEV_PROJECT_ID --billing-account=$BILLING_ACCOUNT_ID
gcloud beta billing projects link $PROD_PROJECT_ID --billing-account=$BILLING_ACCOUNT_ID
```

---

## Phase 2: Configure Shared Project

```powershell
# Switch to Shared Project context
gcloud config set project $SHARED_PROJECT_ID

# Enable APIs
gcloud services enable iam.googleapis.com cloudresourcemanager.googleapis.com iamcredentials.googleapis.com sts.googleapis.com storage.googleapis.com servicenetworking.googleapis.com sqladmin.googleapis.com orgpolicy.googleapis.com

# Create Terraform State Bucket
gcloud storage buckets create gs://$TF_STATE_BUCKET --location=$REGION --uniform-bucket-level-access
gcloud storage buckets update gs://$TF_STATE_BUCKET --versioning
```

---

## Phase 3: Create Service Accounts (in Shared Project)

Service accounts must propagate in GCP before they can be used in IAM bindings. A short delay after creation avoids "Service account does not exist" errors.

```powershell
# Create SAs
gcloud iam service-accounts create $SA_DEV --display-name="GitHub Actions Dev"
gcloud iam service-accounts create $SA_PROD --display-name="GitHub Actions Prod"

# Wait for IAM propagation (avoids "Service account does not exist" when binding immediately)
Start-Sleep -Seconds 10

# Grant Bucket Access (Storage Object Admin)
gcloud storage buckets add-iam-policy-binding gs://$TF_STATE_BUCKET `
    --member="serviceAccount:$SA_DEV@$SHARED_PROJECT_ID.iam.gserviceaccount.com" `
    --role="roles/storage.objectAdmin"

gcloud storage buckets add-iam-policy-binding gs://$TF_STATE_BUCKET `
    --member="serviceAccount:$SA_PROD@$SHARED_PROJECT_ID.iam.gserviceaccount.com" `
    --role="roles/storage.objectAdmin"
```

---

## Phase 4: Configure Workload Identity Federation (WIF)

The workload identity pool and provider may need a moment to propagate before IAM bindings that reference them succeed.

```powershell
# Create Workload Identity Pool
gcloud iam workload-identity-pools create $WIF_POOL_NAME `
    --location="global" `
    --display-name="GitHub Actions Pool"

# Create OIDC Provider
gcloud iam workload-identity-pools providers create-oidc $WIF_PROVIDER_NAME `
    --location="global" `
    --workload-identity-pool=$WIF_POOL_NAME `
    --display-name="GitHub Actions Provider" `
    --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" `
    --attribute-condition="assertion.repository.startsWith('$GITHUB_ORG/')" `
    --issuer-uri="https://token.actions.githubusercontent.com"

# Get the Pool ID for the next step
$POOL_ID = gcloud iam workload-identity-pools describe $WIF_POOL_NAME --location="global" --format="value(name)"

# Wait for WIF pool/provider to propagate before binding to service accounts
Start-Sleep -Seconds 10

# Allow any GitHub repo in the organization to impersonate Dev SA
gcloud iam service-accounts add-iam-policy-binding "$SA_DEV@$SHARED_PROJECT_ID.iam.gserviceaccount.com" `
    --role="roles/iam.workloadIdentityUser" `
    --member="principalSet://iam.googleapis.com/$POOL_ID/attribute.repository/$GITHUB_ORG/*"

# Allow any GitHub repo in the organization to impersonate Prod SA
gcloud iam service-accounts add-iam-policy-binding "$SA_PROD@$SHARED_PROJECT_ID.iam.gserviceaccount.com" `
    --role="roles/iam.workloadIdentityUser" `
    --member="principalSet://iam.googleapis.com/$POOL_ID/attribute.repository/$GITHUB_ORG/*"
```

---

## Phase 5: Create Artifact Registry (in Shared Project)

The Artifact Registry will store Docker images for all environments. It lives in the Shared project so that both dev and prod can pull images from it.

```powershell
# Switch to Shared Project context (if not already there)
gcloud config set project $SHARED_PROJECT_ID

# Enable Artifact Registry API
gcloud services enable artifactregistry.googleapis.com

# Create the Docker repository
gcloud artifacts repositories create org-name-images `
    --repository-format=docker `
    --location=$REGION `
    --description="Docker images for Project Name applications"

# Grant GitHub Actions service accounts admin access to Artifact Registry
# artifactregistry.admin includes all writer permissions (push/pull images) plus IAM management
# This is required for Terraform to grant Cloud Run service accounts access to pull images
# Note: artifactregistry.admin is project-level but scoped to Artifact Registry resources only
$SERVICE_ACCOUNTS = @($SA_DEV, $SA_PROD)
foreach ($sa in $SERVICE_ACCOUNTS) {
    gcloud projects add-iam-policy-binding $SHARED_PROJECT_ID `
        --member="serviceAccount:$sa@$SHARED_PROJECT_ID.iam.gserviceaccount.com" `
        --role="roles/artifactregistry.admin"
}

# Grant GitHub Actions service accounts storage admin access in Shared project
# This is required for Terraform to create GCS buckets (like the client build artifacts bucket)
# storage.admin includes storage.buckets.create permission needed for bucket creation
foreach ($sa in $SERVICE_ACCOUNTS) {
    gcloud projects add-iam-policy-binding $SHARED_PROJECT_ID `
        --member="serviceAccount:$sa@$SHARED_PROJECT_ID.iam.gserviceaccount.com" `
        --role="roles/storage.admin"
}
```

---

## Phase 6: Grant Permissions on Target Projects

We grant the Service Accounts (which live in Shared) permissions on the Target projects.

### Grant Dev Permissions

```powershell
# Enable APIs in Dev Project
gcloud config set project $DEV_PROJECT_ID
gcloud services enable run.googleapis.com sqladmin.googleapis.com compute.googleapis.com servicenetworking.googleapis.com orgpolicy.googleapis.com cloudscheduler.googleapis.com secretmanager.googleapis.com

# Wait for API enablement to propagate before granting roles
Start-Sleep -Seconds 10

# Grant dev permissions
$DEV_ROLES = @(
    "roles/run.admin",
    "roles/cloudsql.admin",
    "roles/storage.admin",
    "roles/compute.networkAdmin",
    "roles/iam.serviceAccountUser",
    "roles/iam.serviceAccountAdmin",
    "roles/iam.securityAdmin",
    "roles/cloudscheduler.admin",
    "roles/secretmanager.admin"
)

foreach ($role in $DEV_ROLES) {
    gcloud projects add-iam-policy-binding $DEV_PROJECT_ID `
        --member="serviceAccount:$SA_DEV@$SHARED_PROJECT_ID.iam.gserviceaccount.com" `
        --role=$role
}

# Allow granting IAM roles to allUsers (public access) in this project
Write-Host "`nSetting policy for $DEV_PROJECT_ID..."
@"
name: projects/$DEV_PROJECT_ID/policies/iam.allowedPolicyMemberDomains
spec:
  rules:
    - allowAll: true
"@ | Out-File -FilePath allow-public-cloudrun-dev.yaml -Encoding UTF8

gcloud org-policies set-policy allow-public-cloudrun-dev.yaml
```

### Grant Prod Permissions

```powershell
# Enable APIs in Prod Project
gcloud config set project $PROD_PROJECT_ID
gcloud services enable run.googleapis.com sqladmin.googleapis.com compute.googleapis.com servicenetworking.googleapis.com orgpolicy.googleapis.com cloudscheduler.googleapis.com secretmanager.googleapis.com

# Wait for API enablement to propagate before granting roles
Start-Sleep -Seconds 10

# Grant prod permissions
$PROD_ROLES = @(
    "roles/run.admin",
    "roles/cloudsql.admin",
    "roles/storage.admin",
    "roles/compute.networkAdmin",
    "roles/iam.serviceAccountUser",
    "roles/iam.serviceAccountAdmin",
    "roles/iam.securityAdmin",
    "roles/cloudscheduler.admin",
    "roles/secretmanager.admin"
)

foreach ($role in $PROD_ROLES) {
    gcloud projects add-iam-policy-binding $PROD_PROJECT_ID `
        --member="serviceAccount:$SA_PROD@$SHARED_PROJECT_ID.iam.gserviceaccount.com" `
        --role=$role
}

# Allow granting IAM roles to allUsers (public access) in this project
Write-Host "`nSetting policy for $PROD_PROJECT_ID..."
@"
name: projects/$PROD_PROJECT_ID/policies/iam.allowedPolicyMemberDomains
spec:
  rules:
    - allowAll: true
"@ | Out-File -FilePath allow-public-cloudrun-prod.yaml -Encoding UTF8

gcloud org-policies set-policy allow-public-cloudrun-prod.yaml
```

---

## Phase 7: Get Values for GitHub Secrets

Run this to print the values you need to copy into GitHub Secrets.

```powershell
Write-Host "=== GITHUB REPOSITORY SECRETS ==="
Write-Host "GCP_WORKLOAD_IDENTITY_PROVIDER: $POOL_ID/providers/$WIF_PROVIDER_NAME"

Write-Host ""
Write-Host "=== GITHUB REPOSITORY VARIABLES ==="
Write-Host "GCP_SHARED_PROJECT_ID: $SHARED_PROJECT_ID"
Write-Host "GCP_TF_STATE_BUCKET: $TF_STATE_BUCKET"
Write-Host "GCP_REGION: $REGION"
Write-Host "APP_NAME: (choose and set your app name - e.g. my-app, it will be used in workflows for my-app-api and my-app-client, etc)"
Write-Host ""
Write-Host "GCP_CLIENT_BUILD_ARTIFACT_BUCKET: (You must choose and set this manually)"
Write-Host "  - Choose a globally unique bucket name (e.g., 'org-name-client-build-artifacts')"
Write-Host "  - This bucket will be created by Terraform in the shared project"
Write-Host "  - Set this as a GitHub Repository Variable with your chosen bucket name"

Write-Host ""
Write-Host "=== GITHUB ENVIRONMENT VARIABLES (for 'dev' environment) ==="
Write-Host "GCP_SERVICE_ACCOUNT: $SA_DEV@$SHARED_PROJECT_ID.iam.gserviceaccount.com"
Write-Host "GCP_ENV_PROJECT_ID: $DEV_PROJECT_ID"

Write-Host ""
Write-Host "=== GITHUB ENVIRONMENT VARIABLES (for 'prod' environment) ==="
Write-Host "GCP_SERVICE_ACCOUNT: $SA_PROD@$SHARED_PROJECT_ID.iam.gserviceaccount.com"
Write-Host "GCP_ENV_PROJECT_ID: $PROD_PROJECT_ID"

Write-Host ""
Write-Host "=== GITHUB ENVIRONMENT SECRETS (for 'dev' environment) ==="
Write-Host "DB_API_PASSWORD: <generate a strong password>"
Write-Host "DB_ADMIN_PASSWORD: <generate a strong password>"

Write-Host ""
Write-Host "=== GITHUB ENVIRONMENT SECRETS (for 'prod' environment) ==="
Write-Host "DB_API_PASSWORD: <generate a different strong password>"
Write-Host "DB_ADMIN_PASSWORD: <generate a different strong password>"
```

**Create the above variables and secrets in GitHub:**

1. **Repository Variables** (Settings → Secrets and variables → Actions → Variables → New repository variable):
   - `GCP_WORKLOAD_IDENTITY_PROVIDER`
   - `GCP_SHARED_PROJECT_ID`
   - `GCP_TF_STATE_BUCKET`
   - `GCP_REGION`
   - `APP_NAME`
   - `GCP_CLIENT_BUILD_ARTIFACT_BUCKET`

3. **Environment Variables** (Settings → Environments → [dev/prod] → Environment variables → Add variable):
   - For `dev` environment: `GCP_SERVICE_ACCOUNT`, `GCP_ENV_PROJECT_ID`
   - For `prod` environment: `GCP_SERVICE_ACCOUNT`, `GCP_ENV_PROJECT_ID`

4. **Environment Secrets** (Settings → Environments → [dev/prod] → Environment secrets → Add secret):
   - For `dev` environment: `DB_API_PASSWORD`, `DB_ADMIN_PASSWORD`
   - For `prod` environment: `DB_API_PASSWORD`, `DB_ADMIN_PASSWORD` (with different values)
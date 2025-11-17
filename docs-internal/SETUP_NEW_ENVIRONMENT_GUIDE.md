# Complete Guide: Setting Up a New Terraform Environment (tomer-env-3)

**Purpose:** Step-by-step instructions for creating a completely isolated new environment from scratch

**Estimated Time:** 3-4 hours (mostly GCP console clicks + waiting for provisioning)

**Assumptions:** You're setting up `tomer-env-3` with no existing environment to copy from

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Phase 1: GCP Project Setup](#phase-1-gcp-project-setup)
3. [Phase 2: OAuth & Authentication Setup](#phase-2-oauth--authentication-setup)
4. [Phase 3: Service Accounts & IAM](#phase-3-service-accounts--iam)
5. [Phase 4: GitHub Integration](#phase-4-github-integration)
6. [Phase 5: Terraform Configuration](#phase-5-terraform-configuration)
7. [Phase 6: Apply Terraform](#phase-6-apply-terraform)
8. [Phase 7: Post-Deployment Setup](#phase-7-post-deployment-setup)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

**Tools Required:**
- `gcloud` CLI installed and authenticated
  ```bash
  gcloud auth login
  gcloud auth application-default login
  ```
- Terraform >= 1.3.0 (check: `terraform version`)
- Git (for cloning repos)
- Text editor (VS Code, nano, vim)
- Access to:
  - Google Cloud Console
  - Microsoft Azure AD (for OAuth)
  - Firebase Console
  - GitHub organization
  - GoDaddy or DNS provider

**Before You Start:**
- Have the project name ready: `tomer-env-3`
- Gather all API keys and secrets (OpenAI, Gemini, HuggingFace, Firebase)
- Have Microsoft OAuth credentials ready
- Know the domain you'll use: `tomer-env-3.thecourtyard.ai`
- Get project owner email: your organization's service account owner

---

## Phase 1: GCP Project Setup

### Step 1.1: Create GCP Project

**In Google Cloud Console:**

1. Go to https://console.cloud.google.com
2. Click **Select a Project** (top left dropdown)
3. Click **NEW PROJECT**
4. Fill in:
   - **Project Name:** `tomer-env-3`
   - **Organization:** Select your organization (if applicable)
   - Click **CREATE**

**Wait 1-2 minutes for project creation...**

```bash
# Verify project was created (CLI)
gcloud projects list | grep tomer-env-3
```

### Step 1.2: Get Project Number

You'll need this for several configurations.

**In Cloud Console:**
1. Navigate to **Select a Project** → `tomer-env-3`
2. Go to **Project Settings** (gear icon, top right)
3. Copy the **Project Number** (looks like: `123456789012`)

**Or via CLI:**
```bash
gcloud projects describe tomer-env-3 --format='value(projectNumber)'
# Output: 123456789012
```

**Save this value** - you'll need it soon for `tomer-env-3.tfvars`

### Step 1.3: Enable Required APIs

**In Cloud Console:**

1. Go to **APIs & Services** → **Library**
2. Search for and **ENABLE** each API:
   - ✅ Compute Engine API
   - ✅ Cloud Run API
   - ✅ Cloud SQL Admin API
   - ✅ Service Networking API
   - ✅ VPC Access API
   - ✅ Secret Manager API
   - ✅ Identity-Aware Proxy API
   - ✅ Cloud DNS API
   - ✅ Network Services API
   - ✅ Firebase Management API
   - ✅ Firebase Authentication API
   - ✅ Cloud Logging API
   - ✅ IAM API
   - ✅ Artifact Registry API
   - ✅ Cloud Build API (for Artifact Registry image building)

**Or via CLI (faster):**
```bash
gcloud services enable \
  compute.googleapis.com \
  run.googleapis.com \
  sqladmin.googleapis.com \
  servicenetworking.googleapis.com \
  vpcaccess.googleapis.com \
  secretmanager.googleapis.com \
  iap.googleapis.com \
  dns.googleapis.com \
  networkservices.googleapis.com \
  identitytoolkit.googleapis.com \
  firebase.googleapis.com \
  logging.googleapis.com \
  iam.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  --project=tomer-env-3
```

**Wait for all APIs to enable (2-3 minutes)**

### Step 1.4: Set Billing Account

**In Cloud Console:**
1. Go to **Billing**
2. Link the project to a billing account (required to create resources)
3. Select your organization's billing account

**If you don't have billing access, contact your GCP administrator**

---

## Phase 2: OAuth & Authentication Setup

### Step 2.1: Create IAP (Identity-Aware Proxy) Branding

**Important:** This must be done ONCE per GCP project.

**In Cloud Console:**

1. Go to **Security** → **Identity-Aware Proxy**
2. On the **OAuth Consent Screen** tab:
   - Click **EDIT APP**
   - Fill in:
     - **App name:** `Thecourtyard AI - tomer-env-3`
     - **User support email:** Your email (e.g., `tomer@thecourtyard.ai`)
     - **Developer contact info:** Your email
     - Click **SAVE AND CONTINUE**

3. On **Scopes** tab:
   - Click **ADD OR REMOVE SCOPES**
   - Add: `openid`, `email`, `profile`
   - Click **UPDATE**
   - Click **SAVE AND CONTINUE**

4. On **Test Users** tab:
   - Add yourself as test user
   - Click **SAVE AND CONTINUE**

**Or via CLI:**
```bash
# Get the brand (after created in console)
gcloud iap oauth-brands list --project=tomer-env-3
```

### Step 2.2: Create IAP OAuth Client

**In Cloud Console:**

1. Go to **Security** → **Identity-Aware Proxy**
2. Go to **Applications** tab (top menu)
3. Click **CREATE OAUTH CLIENT**
4. Choose type: **Web application**
5. Fill in:
   - **Name:** `Terraform IAP Client`
   - **Authorized JavaScript origins:**
     - `https://tomer-env-3.thecourtyard.ai`
   - **Authorized redirect URIs:**
     - `https://tomer-env-3.thecourtyard.ai/`
     - `https://tomer-env-3.thecourtyard.ai/_gcp_gatekeeper/authenticate_oauth`
   - Click **CREATE**

6. **IMPORTANT:** Copy and save:
   - **Client ID**
   - **Client Secret**

**You'll need these values in `tomer-env-3.tfvars`**

```bash
# Verify via CLI
gcloud iap oauth-clients list \
  --project=tomer-env-3 \
  --filter="displayName:Terraform" \
  --format="value(name,clientId)"
```

### Step 2.3: Create Microsoft OAuth App

**In Azure Portal:**

1. Go to https://portal.azure.com
2. Navigate to **Azure Active Directory** → **App registrations**
3. Click **+ New registration**
4. Fill in:
   - **Name:** `Thecourtyard AI - tomer-env-3`
   - **Supported account types:** Select "Accounts in any organizational directory (Any Microsoft Entra ID tenant - Multitenant)"
   - Click **REGISTER**

5. On the new app page:
   - **Copy Application (client) ID** - save this
   - Go to **Authentication** → **+ Add a platform**
   - Choose **Web**
   - Fill in **Redirect URIs:**
     - `https://tomer-env-3.thecourtyard.ai`
     - `http://localhost:5174`
     - `https://127.0.0.1:5174`
     - `https://<FIREBASE_PROJECT>.firebaseapp.com/__/auth/handler`
     - `https://<FIREBASE_PROJECT>.firebaseapp.com`
   - Click **Configure**

6. Go to **Certificates & secrets** → **Client secrets**
   - Click **+ New client secret**
   - **Description:** `Terraform`
   - **Expires:** Select `24 months`
   - Click **Add**
   - **IMPORTANT:** Copy the secret VALUE immediately (you won't see it again)

7. Go to **API permissions**
   - Click **+ Add a permission**
   - Choose **Microsoft Graph**
   - Select **Delegated permissions**
   - Search for and select:
     - `email`
     - `profile`
     - `User.Read`
     - `openid`
   - Click **Add permissions**

8. Go to **Token configuration**
   - Click **+ Add optional claim**
   - Choose **ID** token type
   - Add claims:
     - ✅ `auth_time`
     - ✅ `email`
     - ✅ `family_name`
     - ✅ `given_name`
     - ✅ `groups`
     - ✅ `preferred_username`
     - ✅ `upn`
   - Click **Add**

**Save the following values:**
```
MICROSOFT_CLIENT_ID = <Application (client) ID>
MICROSOFT_CLIENT_SECRET = <Secret value>
```

### Step 2.4: Create Firebase Project

**In Firebase Console:**

1. Go to https://console.firebase.google.com
2. Click **+ Create project**
3. **Project name:** `tomer-env-3`
4. **Enable Google Analytics:** (optional, uncheck if not needed)
5. Click **CREATE PROJECT**

**Wait for Firebase to initialize (2-3 minutes)**

6. Once created, go to **Project settings** (gear icon)
7. Go to **Service accounts** tab
8. Click **Generate new private key**
9. This downloads a JSON file: `tomer-env-3-firebase-adminsdk-*.json`

**Convert Firebase key to base64:**
```bash
# On macOS/Linux:
base64 -i tomer-env-3-firebase-adminsdk-*.json | tr -d '\n' > firebase-key-base64.txt

# On Windows PowerShell:
[Convert]::ToBase64String([System.IO.File]::ReadAllBytes("tomer-env-3-firebase-adminsdk-*.json")) | Set-Content firebase-key-base64.txt

# On Windows (Git Bash):
openssl base64 -in tomer-env-3-firebase-adminsdk-*.json | tr -d '\n' > firebase-key-base64.txt
```

**Save the base64 output** - you'll need it for `tomer-env-3.tfvars`

**Also go to Firebase Authentication:**
1. **Authentication** → **Sign-in method**
2. Enable providers:
   - ✅ Google
   - ✅ Microsoft

---

## Phase 3: Service Accounts & IAM

### Step 3.1: Create Terraform Service Account

**In Cloud Console:**

1. Go to **IAM & Admin** → **Service Accounts**
2. Click **+ CREATE SERVICE ACCOUNT**
3. Fill in:
   - **Service account name:** `terraform-sa`
   - **Service account ID:** `terraform-sa` (auto-filled)
   - **Description:** `Terraform service account for infrastructure deployment`
   - Click **CREATE AND CONTINUE**

4. **Grant roles** (in "Grant this service account access to project"):
   - Click **+ GRANT ROLES**
   - Add each role:
     - ✅ `Editor` (roles/editor)
     - ✅ `Service Account Admin` (roles/iam.serviceAccountAdmin)
     - ✅ `Service Account User` (roles/iam.serviceAccountUser)
     - ✅ `Service Usage Admin` (roles/serviceusage.serviceUsageAdmin)
   - Click **CONTINUE**
   - Click **DONE**

**Or via CLI:**
```bash
# Create service account
gcloud iam service-accounts create terraform-sa \
  --description="Terraform service account for infrastructure deployment" \
  --display-name="Terraform SA" \
  --project=tomer-env-3

# Grant roles
gcloud projects add-iam-policy-binding tomer-env-3 \
  --member="serviceAccount:terraform-sa@tomer-env-3.iam.gserviceaccount.com" \
  --role="roles/editor"

gcloud projects add-iam-policy-binding tomer-env-3 \
  --member="serviceAccount:terraform-sa@tomer-env-3.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountAdmin"

gcloud projects add-iam-policy-binding tomer-env-3 \
  --member="serviceAccount:terraform-sa@tomer-env-3.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding tomer-env-3 \
  --member="serviceAccount:terraform-sa@tomer-env-3.iam.gserviceaccount.com" \
  --role="roles/serviceusage.serviceUsageAdmin"
```

### Step 3.2: Create Terraform Service Account Key

This is the key you'll use for local Terraform execution.

**In Cloud Console:**

1. Go to **IAM & Admin** → **Service Accounts**
2. Click on `terraform-sa@tomer-env-3.iam.gserviceaccount.com`
3. Go to **Keys** tab
4. Click **+ CREATE KEY**
5. Choose **JSON**
6. Click **CREATE**

**This downloads `tomer-env-3-terraform-key.json`**

**Use this key to authenticate locally:**
```bash
# Set up gcloud to use this key
gcloud auth activate-service-account --key-file=tomer-env-3-terraform-key.json

# Verify it works
gcloud auth list
gcloud projects list
```

### Step 3.3: Grant User IAP Permissions (Optional but Recommended)

This allows you to impersonate the terraform SA to test/debug.

**In Cloud Console:**

1. Go to **IAM & Admin** → **IAM**
2. Click **+ GRANT ACCESS**
3. **New principal:** Your email (e.g., `tomer@thecourtyard.ai`)
4. **Roles:** Select `Service Account Token Creator` (roles/iam.serviceAccountTokenCreator)
5. Click **SAVE**

**Or via CLI:**
```bash
gcloud iam service-accounts add-iam-policy-binding \
  terraform-sa@tomer-env-3.iam.gserviceaccount.com \
  --member="user:tomer@thecourtyard.ai" \
  --role="roles/iam.serviceAccountTokenCreator" \
  --project=tomer-env-3
```

---

## Phase 4: GitHub Integration

### Step 4.1: Create GitHub Workload Identity Federation Pool

This allows GitHub Actions to authenticate to GCP WITHOUT storing service account keys.

**In Cloud Console:**

1. Go to **IAM & Admin** → **Workload Identity Federation**
2. Click **+ CREATE WORKLOAD IDENTITY POOL**
3. Fill in:
   - **Pool ID:** `github-pool`
   - **Pool display name:** `GitHub Workload Identity Pool`
   - **Location:** Select `Global`
   - Click **CONTINUE**

4. **Configure provider:**
   - **Provider ID:** `github-provider`
   - **Provider display name:** `GitHub OIDC Provider`
   - **OpenID Connect (OIDC) provider URL:** `https://token.actions.githubusercontent.com`
   - Click **CONTINUE**

5. **Map Google Cloud attributes:**
   - Click **ADD MAPPING**
   - **Google Cloud attribute:** `google.subject`
   - **OIDC attribute:** `assertion.sub`
   - Click **ADD MAPPING** again
   - **Google Cloud attribute:** `attribute.repository`
   - **OIDC attribute:** `assertion.repository`
   - Click **SAVE**

6. **Configure attribute conditions (optional but recommended):**
   - Add: `assertion.repository.startsWith('courtyard-ai/')`
   - This restricts access to repos in your org
   - Click **SAVE**

**Or via CLI:**
```bash
# Create pool
gcloud iam workload-identity-pools create "github-pool" \
  --location="global" \
  --display-name="GitHub Workload Identity Pool" \
  --project=tomer-env-3

# Create provider
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub OIDC Provider" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository.startsWith('courtyard-ai/')" \
  --project=tomer-env-3
```

### Step 4.2: Create GitHub Deployer Service Account

**In Cloud Console:**

1. Go to **IAM & Admin** → **Service Accounts**
2. Click **+ CREATE SERVICE ACCOUNT**
3. Fill in:
   - **Service account name:** `github-deployer`
   - **Service account ID:** `github-deployer` (auto-filled)
   - **Description:** `Used by GitHub Actions to deploy to GCP`
   - Click **CREATE AND CONTINUE**

4. **Grant roles:**
   - Click **+ GRANT ROLES**
   - Add roles:
     - ✅ `Cloud Run Admin` (roles/run.admin)
     - ✅ `Storage Admin` (roles/storage.admin)
     - ✅ `Artifact Registry Reader` (roles/artifactregistry.reader)
     - ✅ `Artifact Registry Writer` (roles/artifactregistry.writer)
     - ✅ `Service Account User` (roles/iam.serviceAccountUser)
   - Click **CONTINUE**
   - Click **DONE**

**Or via CLI:**
```bash
# Create service account
gcloud iam service-accounts create github-deployer \
  --description="Used by GitHub Actions to deploy to GCP" \
  --display-name="GitHub Deployer" \
  --project=tomer-env-3

# Grant roles
for role in roles/run.admin roles/storage.admin roles/artifactregistry.reader roles/artifactregistry.writer roles/iam.serviceAccountUser; do
  gcloud projects add-iam-policy-binding tomer-env-3 \
    --member="serviceAccount:github-deployer@tomer-env-3.iam.gserviceaccount.com" \
    --role="$role"
done
```

### Step 4.3: Grant GitHub OIDC Access to Deploy Service Account

This binds the GitHub OIDC provider to the deployer SA so GHA can authenticate.

**In Cloud Console:**

1. Go to **IAM & Admin** → **Service Accounts**
2. Click `github-deployer@tomer-env-3.iam.gserviceaccount.com`
3. Go to **Permissions** tab
4. Click **+ GRANT ACCESS**
5. **New principal:** (for EACH repo that will deploy)
   ```
   principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/courtyard-ai/lawgic-ai-backend
   ```
   Replace `PROJECT_NUMBER` with your project number, and `lawgic-ai-backend` with your repo name

6. **Role:** `Workload Identity User` (roles/iam.workloadIdentityUser)
7. Click **SAVE**

**Repeat for each GitHub repo that will deploy:**
- `courtyard-ai/lawgic-ai-backend`
- `courtyard-ai/lawgic-ai-engine`
- `courtyard-ai/lawgic-ai-dashboard`

**Or via CLI:**
```bash
PROJECT_NUMBER=$(gcloud projects describe tomer-env-3 --format='value(projectNumber)')

# For each repo:
gcloud iam service-accounts add-iam-policy-binding \
  github-deployer@tomer-env-3.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/courtyard-ai/lawgic-ai-backend" \
  --project=tomer-env-3

# Repeat for engine and dashboard repos...
```

### Step 4.4: Get GitHub WIF Details (For Later Use)

You'll need these values for GitHub Actions workflow files.

```bash
PROJECT_NUMBER=$(gcloud projects describe tomer-env-3 --format='value(projectNumber)')

echo "Service Account Email: github-deployer@tomer-env-3.iam.gserviceaccount.com"
echo "Workload Identity Provider: projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/providers/github-provider"
```

**Save these values** - you'll need them in GitHub Actions workflows

---

## Phase 5: Terraform Configuration

### Step 5.1: Prepare Environment Variables (Secrets)

You'll need these values for `tomer-env-3.tfvars`. Gather:

**From GCP (already created):**
- ✅ `project_id`: `tomer-env-3`
- ✅ `project_number`: (from Step 1.2)
- ✅ `google_iap_client_id`: (from Step 2.2)
- ✅ `google_iap_client_secret`: (from Step 2.2)

**From Microsoft Azure:**
- ✅ `microsoft_client_id`: (from Step 2.3)
- ✅ `microsoft_client_secret`: (from Step 2.3)

**From Firebase:**
- ✅ `firebase_service_account_key_base64`: (from Step 2.4)

**Generated (create secure passwords):**
```bash
# Generate secure password for database
openssl rand -base64 32
# Output: abc123... (save as DB_PASSWORD)

# Generate secure password for admin user
openssl rand -base64 32
# Output: xyz789... (save as ADMIN_PASSWORD)
```

**From Third-party APIs (gather these separately):**
- ✅ `openai_api_key`: Get from OpenAI dashboard
- ✅ `gemini_api_key`: Get from Google Cloud's Vertex AI
- ✅ `huggingface_api_token`: Get from HuggingFace account

**Organization/User Info:**
- ✅ `workspace_user_email`: Your email (e.g., `tomer@thecourtyard.ai`)
- ✅ `admin_user_email`: Admin email (can be same as yours)
- ✅ `domain`: `tomer-env-3.thecourtyard.ai` (or your domain)

### Step 5.2: Create tfvars File

Navigate to the Terraform directory:

```bash
cd c:/Users/tomer/dev2/devops/infra/terraform/gcp
```

Create `tomer-env-3.tfvars`:

```bash
cat > tomer-env-3.tfvars << 'EOF'
# Project Configuration
project_id     = "tomer-env-3"
project_number = "123456789012"  # FROM STEP 1.2
region         = "me-west1"
zone           = "me-west1-a"

# Environment Identification
environment          = "tomer-env-3"
workspace_user_email = "tomer@thecourtyard.ai"

# Database Configuration
db_user     = "courtyard"
db_password = "PASTE_DB_PASSWORD_HERE"  # FROM STEP 5.1
db_name     = "courtyard"

# Cloud Run Services
backend_service_name   = "backend-api"
backend_repository_id  = "development"
backend_image_id       = "backend"

engine_service_name    = "engine"
engine_repository_id   = "development"
engine_image_id        = "engine"

dashboard_service_name = "dashboard"
dashboard_repository_id = "development"
dashboard_image_id      = "dashboard"

# Storage
dashboard_bucket_name = "tomer-env-3-dashboard-bucket"

# Domain & Support
domain        = "tomer-env-3.thecourtyard.ai"
support_email = "tomer@thecourtyard.ai"

# OAuth & Authentication
microsoft_client_id     = "PASTE_MICROSOFT_CLIENT_ID_HERE"          # FROM STEP 2.3
microsoft_client_secret = "PASTE_MICROSOFT_CLIENT_SECRET_HERE"      # FROM STEP 2.3
admin_user_email        = "tomer@thecourtyard.ai"
admin_user_password     = "PASTE_ADMIN_PASSWORD_HERE"               # FROM STEP 5.1

# AI Services
openai_api_key           = "PASTE_OPENAI_API_KEY_HERE"
gemini_api_key           = "PASTE_GEMINI_API_KEY_HERE"
huggingface_api_token    = "PASTE_HUGGINGFACE_TOKEN_HERE"
firebase_service_account_key_base64 = "PASTE_FIREBASE_KEY_BASE64_HERE"  # FROM STEP 2.4

# Cloud Location
google_cloud_location = "us-central1"
EOF
```

### Step 5.3: Update tfvars with Actual Values

**Edit the file with your values:**

```bash
# On Windows:
notepad tomer-env-3.tfvars

# On macOS/Linux:
nano tomer-env-3.tfvars
```

**Replace all `PASTE_*` placeholders with actual values from Step 5.1**

**IMPORTANT - Secure handling:**
- ❌ DON'T commit tfvars with real secrets to Git
- ✅ DO store in 1Password, LastPass, or `.gitignore`
- ✅ DO use environment variables in CI/CD

### Step 5.4: Create Terraform Workspace

```bash
# List existing workspaces
terraform workspace list

# Create new workspace
terraform workspace new tomer-env-3

# Verify workspace was created
terraform workspace list
# Output:
#   default
# * tomer-env-3
```

### Step 5.5: Initialize Terraform Backend

```bash
# Initialize Terraform
terraform init

# Verify initialization succeeded
terraform validate
# Output: Success! The configuration is valid.
```

---

## Phase 6: Apply Terraform

### Step 6.1: Plan Terraform Changes

```bash
# Generate plan
terraform plan -var-file="tomer-env-3.tfvars" -out=tomer-env-3.tfplan

# Review output - should show:
# - Network resources (VPC, subnets)
# - PostgreSQL database
# - Service accounts
# - Cloud Run services (backend, engine, dashboard)
# - Load balancers
# - Secret Manager secrets
# - etc.

# You should see: Plan: XXX to add, 0 to change, 0 to destroy
```

**If there are errors, see Troubleshooting section**

### Step 6.2: Apply Terraform Configuration

```bash
# Apply the plan
terraform apply tomer-env-3.tfplan

# Wait for infrastructure to be created (5-10 minutes)
# Watch the output for any errors
```

**What Terraform is Creating:**
- VPC network with 3 subnets
- PostgreSQL database instance
- 3 Cloud Run services (backend, engine, dashboard)
- 3 Artifact Registry repositories
- Secret Manager secrets
- Load balancer with SSL
- Internal DNS zone
- Security policies
- Debug VM (optional)

### Step 6.3: Verify Terraform Succeeded

```bash
# Check Terraform state
terraform state list
# Should show all created resources

# Verify resources in GCP
gcloud compute networks list --project=tomer-env-3
gcloud sql instances list --project=tomer-env-3
gcloud run services list --project=tomer-env-3
gcloud compute instances list --project=tomer-env-3
```

---

## Phase 7: Post-Deployment Setup

### Step 7.1: DNS Configuration

You need to create DNS records for your domain to point to the load balancer IP.

**Get the Load Balancer IP:**

```bash
# Find the external IP
gcloud compute addresses list --project=tomer-env-3

# Or from Terraform output (if configured)
terraform output -var-file="tomer-env-3.tfvars"
```

**Look for:** `tomer-env-3-lb-ip` (should be a global static IP like `1.2.3.4`)

**In GoDaddy or your DNS provider:**

1. Go to DNS settings for `thecourtyard.ai`
2. Create an A record:
   - **Host:** `tomer-env-3`
   - **Points to:** `1.2.3.4` (the load balancer IP)
   - **TTL:** `3600` (1 hour) or auto
   - Click **Save**

3. Verify DNS resolves:
   ```bash
   nslookup tomer-env-3.thecourtyard.ai
   # Should return the load balancer IP
   ```

### Step 7.2: Update Artifact Registry Repositories

The Terraform created 3 empty Artifact Registry repositories. You need to push initial Docker images.

**For each service (backend, engine, dashboard), update their GitHub workflow files:**

1. Go to the repository (e.g., `lawgic-ai-backend`)
2. Go to `.github/workflows/deploy.yml`
3. Update the image location:
   ```yaml
   - name: Deploy to Cloud Run
     uses: google-github-actions/deploy-cloudrun@v0
     with:
       service: tomer-env-3-backend-api
       region: me-west1
       image: me-west1-docker.pkg.dev/tomer-env-3/development/backend:latest
   ```

**Or push initial placeholder image:**
```bash
# Authenticate Docker with Artifact Registry
gcloud auth configure-docker me-west1-docker.pkg.dev

# Build and push a placeholder image
docker build -t me-west1-docker.pkg.dev/tomer-env-3/development/backend:latest .
docker push me-west1-docker.pkg.dev/tomer-env-3/development/backend:latest
```

### Step 7.3: Create GitHub Actions Secrets

For each repository (`lawgic-ai-backend`, `lawgic-ai-engine`, `lawgic-ai-dashboard`):

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Create secrets:

```
GCP_PROJECT_ID = tomer-env-3
GCP_REGION = me-west1
GCP_WORKLOAD_IDENTITY_PROVIDER = projects/123456789012/locations/global/workloadIdentityPools/github-pool/providers/github-provider
GCP_SERVICE_ACCOUNT = github-deployer@tomer-env-3.iam.gserviceaccount.com
```

**Get these from Step 4.4**

### Step 7.4: Create/Update GitHub Workflows

Each repository needs a GitHub Actions workflow file to deploy to the new environment.

**Example `.github/workflows/deploy-tomer-env-3.yml`:**

```yaml
name: Deploy to tomer-env-3

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v1
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v1

      - name: Build and push Docker image
        run: |
          gcloud builds submit \
            --tag me-west1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/development/backend:latest \
            --project=${{ secrets.GCP_PROJECT_ID }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy tomer-env-3-backend-api \
            --image me-west1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/development/backend:latest \
            --region ${{ secrets.GCP_REGION }} \
            --project=${{ secrets.GCP_PROJECT_ID }}
```

### Step 7.5: Test Load Balancer Access

```bash
# Test the external load balancer
curl -I https://tomer-env-3.thecourtyard.ai/health
# Should return: HTTP/2 200 (or 403 if IP is blocked)

# Test internal services
gcloud compute instances ssh tomer-env-3-debug-vm \
  --zone=me-west1-a \
  --project=tomer-env-3 \
  --command="curl http://engine.tomer-env-3.internal:8888/health"
```

### Step 7.6: Verify Database Connection

```bash
# SSH into debug VM
gcloud compute instances ssh tomer-env-3-debug-vm \
  --zone=me-west1-a \
  --project=tomer-env-3

# From debug VM, test database:
apt update && apt install -y postgresql-client
psql -h 10.0.0.3 -U courtyard -d courtyard -c "SELECT version();"
# Enter password: (from tomer-env-3.tfvars db_password)
```

### Step 7.7: Monitor Terraform Outputs

```bash
# Get IAP OAuth credentials
terraform output -json

# Expected output:
# {
#   "iap_oauth_client_id": "123...xyz.apps.googleusercontent.com",
#   "iap_oauth_client_secret": "GOCSPX-..."
# }
```

---

## Troubleshooting

### Issue: `terraform init` fails with backend error

**Error:** `Error: Backend initialization required`

**Solution:**
```bash
# Check credentials
gcloud auth list
# Output should show: terraform-sa@tomer-env-3.iam.gserviceaccount.com (active)

# If not, re-authenticate:
gcloud auth activate-service-account --key-file=tomer-env-3-terraform-key.json
gcloud auth application-default login
```

---

### Issue: APIs not enabled error

**Error:** `Permission denied on service ... API`

**Solution:**
```bash
# Re-enable all required APIs
gcloud services enable \
  compute.googleapis.com \
  run.googleapis.com \
  sqladmin.googleapis.com \
  --project=tomer-env-3
```

---

### Issue: VPC Peering fails during `terraform apply`

**Error:** `Error: Error creating Service Networking Connection`

**Solution:**
Usually resolves itself after retry:
```bash
# Wait 2 minutes, then retry
terraform apply -var-file="tomer-env-3.tfvars"
```

---

### Issue: Cloud Run services in `Error` state

**Error:** Pod stays in `Pending` state or Error creating service

**Check logs:**
```bash
gcloud run services describe tomer-env-3-backend-api \
  --region=me-west1 \
  --project=tomer-env-3 \
  --format=yaml
```

**Common causes:**
- ❌ Docker image not found (Artifact Registry empty)
- ❌ VPC connector not ready (wait 5 minutes)
- ❌ Secrets not accessible (check IAM)

---

### Issue: Load balancer returns 403 Forbidden

**Cause:** IP whitelist security policy is blocking your IP

**Solution:**

Option A: Check your IP and add it to `main.tf` (lines 437, 464):
```bash
# Get your IP
curl ifconfig.me
# Output: 1.2.3.4

# Add to security policy in main.tf:
src_ip_ranges = ["1.2.3.4/32", "147.236.179.64/32", "..."]

# Re-apply:
terraform apply -var-file="tomer-env-3.tfvars"
```

Option B: Temporarily disable security policy (testing only):
```bash
# Temporarily allow all traffic
gcloud compute security-policies rules update 2147483647 \
  --action allow \
  --project=tomer-env-3 \
  --security-policy=tomer-env-3-ip-whitelist-policy

# Test access

# Re-apply security policy:
terraform apply -var-file="tomer-env-3.tfvars"
```

---

### Issue: Database connection times out

**Error:** `timeout waiting for connection from 10.0.0.3:5432`

**Cause:** VPC Access Connector not yet ready

**Solution:**
```bash
# Wait 5-10 minutes for connector to initialize, then test again

# Check connector status:
gcloud compute vpc-access connectors list \
  --region=me-west1 \
  --project=tomer-env-3
```

---

### Issue: Secret Manager values not found

**Error:** `Error 404: Secret not found`

**Solution:**
```bash
# Verify secrets were created
gcloud secrets list --project=tomer-env-3

# Verify Cloud Run has access
gcloud secrets get-iam-policy tomer-env-3-backend-env \
  --project=tomer-env-3
```

---

### Issue: DNS not resolving

**Error:** `Name or service not known` when pinging domain

**Solution:**
```bash
# Verify DNS record was created
nslookup tomer-env-3.thecourtyard.ai

# If not resolving, check:
# 1. DNS record exists in GoDaddy
# 2. Load balancer IP is correct
# 3. Wait 5-10 minutes for DNS propagation

# Force DNS refresh:
ipconfig /flushdns  # Windows
sudo dscacheutil -flushcache  # macOS
sudo systemctl restart systemd-resolved  # Linux
```

---

## Next Steps After Setup

1. **Deploy applications** - Push code to GitHub, workflows deploy to Cloud Run
2. **Monitor resources** - Set up Cloud Monitoring/Logging dashboards
3. **Test end-to-end** - Verify all 3 services communicate
4. **Set up backups** - Ensure database backups are scheduled
5. **Configure alerts** - Set up alert policies for errors/high latency
6. **Document credentials** - Store all secrets in 1Password/vault
7. **Team access** - Grant IAM roles to team members

---

## Checklist

**Pre-Setup:**
- [ ] GCP Organization access
- [ ] Azure AD access (for Microsoft OAuth)
- [ ] Firebase access
- [ ] GitHub organization admin access
- [ ] GoDaddy/DNS provider access
- [ ] API keys ready (OpenAI, Gemini, HuggingFace)

**Phase 1: GCP Project**
- [ ] Project created (`tomer-env-3`)
- [ ] Project number saved
- [ ] 15 APIs enabled
- [ ] Billing account linked

**Phase 2: OAuth**
- [ ] IAP branding created
- [ ] IAP OAuth client created (ID + secret saved)
- [ ] Microsoft app registered (ID + secret saved)
- [ ] Firebase project created
- [ ] Firebase key in base64 saved

**Phase 3: Service Accounts**
- [ ] `terraform-sa` created with 4 roles
- [ ] `terraform-sa` key downloaded and saved
- [ ] `github-deployer` created with 5 roles
- [ ] User IAP permissions granted

**Phase 4: GitHub**
- [ ] WIF pool created (`github-pool`)
- [ ] WIF provider created (`github-provider`)
- [ ] `github-deployer` SA has WIF permissions
- [ ] WIF details saved (service account email + provider path)

**Phase 5: Terraform**
- [ ] `tomer-env-3.tfvars` created with all values
- [ ] `tomer-env-3` workspace created
- [ ] `terraform init` succeeded
- [ ] `terraform validate` passed

**Phase 6: Apply**
- [ ] `terraform plan` reviewed
- [ ] `terraform apply` succeeded
- [ ] All resources created (check `terraform state list`)

**Phase 7: Post-Deployment**
- [ ] DNS record created and resolving
- [ ] Load balancer accessible
- [ ] Artifact Registry repos ready
- [ ] GitHub Actions secrets configured
- [ ] GitHub workflows created
- [ ] Database connectivity verified

---

## Success Criteria

Your environment is ready when:

✅ `curl https://tomer-env-3.thecourtyard.ai/health` returns `200 OK`
✅ `terraform state list` shows all resources created
✅ `gcloud sql instances list` shows PostgreSQL database
✅ `gcloud run services list` shows 3 Cloud Run services (backend, engine, dashboard)
✅ `gcloud container images list` shows 3 image repos (backend, engine, dashboard)
✅ GitHub Actions can deploy to Cloud Run
✅ Database password works and you can query the database
✅ All 3 services can communicate via internal load balancer

---

**Congratulations! You now have a fully isolated `tomer-env-3` environment!** 🎉

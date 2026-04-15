# CI/CD Pipeline for React App on Azure App Service using GitHub Actions

A step-by-step guide for setting up an automated build-and-deploy pipeline that publishes a React application to Azure App Service every time code is pushed to GitHub.

---

## Table of Contents

1. [What This Guide Covers](#1-what-this-guide-covers)
2. [Key Concepts (for Beginners)](#2-key-concepts-for-beginners)
3. [Prerequisites](#3-prerequisites)
4. [Architecture Overview](#4-architecture-overview)
5. [Step 1: Create an Azure App Service](#step-1-create-an-azure-app-service)
6. [Step 2: Create a GitHub Repository](#step-2-create-a-github-repository)
7. [Step 3: Create an Azure AD App Registration](#step-3-create-an-azure-ad-app-registration)
8. [Step 4: Set Up OIDC Federation (Passwordless Auth)](#step-4-set-up-oidc-federation-passwordless-auth)
9. [Step 5: Assign Azure Permissions](#step-5-assign-azure-permissions)
10. [Step 6: Configure GitHub Secrets and Variables](#step-6-configure-github-secrets-and-variables)
11. [Step 7: Create a GitHub Environment](#step-7-create-a-github-environment)
12. [Step 8: Understand the Pipeline](#step-8-understand-the-pipeline)
13. [Step 9: Push Code and Trigger the Pipeline](#step-9-push-code-and-trigger-the-pipeline)
14. [Step 10: Monitor and Troubleshoot](#step-10-monitor-and-troubleshoot)
15. [Common Errors and Fixes](#common-errors-and-fixes)
16. [Security Best Practices](#security-best-practices)
17. [Appendix: GitHub Actions Explained](#appendix-github-actions-explained)

---

## 1. What This Guide Covers

This guide walks you through building a CI/CD pipeline that:

- **Automatically builds** your React app (compiles JSX → static HTML/CSS/JS) on every push or pull request
- **Automatically deploys** the built app to Azure App Service when code is merged to the `main` branch
- **Uses passwordless authentication** (OIDC) — no passwords or API keys stored anywhere
- **Runs on GitHub-hosted runners** — free, managed virtual machines that GitHub spins up on demand

---

## 2. Key Concepts (for Beginners)

### GitHub Concepts

| Concept | What It Is | Analogy |
|---------|------------|---------|
| **Repository (Repo)** | A folder that stores your code with full version history | A shared network drive that remembers every change |
| **Branch** | A parallel version of your code. `main` is the primary branch | A draft copy you can edit without affecting the original |
| **Push** | Uploading your local code changes to GitHub | Saving a file to the shared drive |
| **Pull Request (PR)** | A request to merge changes from one branch into another | Asking a colleague to review and approve your edits |
| **Commit** | A snapshot of your code at a point in time | A "Save As" with a description of what changed |

### CI/CD Concepts

| Concept | What It Is | Analogy |
|---------|------------|---------|
| **CI (Continuous Integration)** | Automatically building and testing code on every change | A spell-checker that runs every time you save |
| **CD (Continuous Deployment)** | Automatically deploying tested code to production | Auto-publishing a document after it passes review |
| **GitHub Actions** | GitHub's built-in CI/CD platform | An assistant that follows a recipe when you save a file |
| **Workflow** | A YAML file defining automation steps | The recipe itself |
| **Job** | A group of steps running on the same machine | A chapter in the recipe |
| **Step** | A single command or action | One instruction in the recipe |
| **Runner** | The machine (VM) that executes your workflow | The kitchen where the recipe is cooked |
| **GitHub-hosted Runner** | A free, temporary VM managed by GitHub | A rental kitchen that's cleaned after every use |
| **Artifact** | A file saved between jobs (since each job runs on a separate VM) | A takeout container to move food between kitchens |

### Azure Concepts

| Concept | What It Is |
|---------|------------|
| **App Service** | Azure's managed web hosting — runs your app without managing servers |
| **App Service Plan** | The VM size/tier that hosts your App Service (F1 = free tier) |
| **Resource Group** | A logical container that groups related Azure resources |
| **OIDC** | OpenID Connect — lets GitHub prove its identity to Azure without passwords |
| **Service Principal** | An identity (like a service account) that applications use to access Azure |

---

## 3. Prerequisites

### Azure

- [ ] An Azure subscription (free tier works)
- [ ] Azure CLI installed — [Install Guide](https://learn.microsoft.com/cli/azure/install-azure-cli)
  - Verify: `az --version`

### GitHub

- [ ] A GitHub account
- [ ] Git installed locally
  - Verify: `git --version`
- [ ] GitHub CLI installed — [Install Guide](https://cli.github.com/)
  - Verify: `gh --version`

### Local Machine

- [ ] Node.js 18+ installed — [Download](https://nodejs.org/)
  - Verify: `node --version` and `npm --version`

---

## 4. Architecture Overview

### End-to-End Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                          YOUR MACHINE                               │
│  1. Write React code (components, pages, styles)                    │
│  2. git push → sends code to GitHub                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           GITHUB                                    │
│                                                                     │
│  3. GitHub detects the push                                         │
│  4. Reads .github/workflows/deploy-react-app.yml                    │
│                                                                     │
│  ┌─── Runner VM #1 (build job) ──────────────────────────────┐      │
│  │  5. Checks out your code                                  │      │
│  │  6. Installs Node.js 22                                   │      │
│  │  7. Runs npm ci (install dependencies)                    │      │
│  │  8. Runs npm run build (React → static HTML/CSS/JS)       │      │
│  │  9. Uploads dist/ folder as an artifact                   │      │
│  └───────────────────────────────────────────────────────────┘      │
│          │ artifact (dist/ folder)                                   │
│          ▼                                                          │
│  ┌─── Runner VM #2 (deploy job) ─────────────────────────────┐      │
│  │ 10. Downloads the dist/ artifact                          │      │
│  │ 11. OIDC login → gets Azure access token                  │      │
│  │ 12. Deploys dist/ to Azure App Service                    │      │
│  └───────────────────────────────────────────────────────────┘      │
│                                                                     │
│ 13. Both VMs are destroyed (nothing persists)                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AZURE APP SERVICE                              │
│                                                                     │
│  Your React app is live at https://<app-name>.azurewebsites.net     │
└─────────────────────────────────────────────────────────────────────┘
```

### Why Two Separate VMs?

Each job runs on a **fresh, isolated VM**. This provides:

- **Security** — the build VM never has Azure credentials
- **Flexibility** — build runs on every PR (for validation), deploy only on `main`
- **Approval gates** — you can require manual approval before deploy starts

Since the two VMs don't share a filesystem, we use **artifacts** to pass the built files from the build VM to the deploy VM.

### How OIDC Authentication Works

```
GitHub Runner                    Azure AD                      Azure App Service
     │                              │                               │
     │  1. "I am repo X,            │                               │
     │      environment production"  │                               │
     │  ────────────────────────────►│                               │
     │                              │                               │
     │  2. "Here's a short-lived    │                               │
     │      access token"           │                               │
     │  ◄────────────────────────────│                               │
     │                              │                               │
     │  3. "Deploy these files"     │                               │
     │      (with Azure token)      │                               │
     │  ────────────────────────────────────────────────────────────►│
     │                              │                               │
     │  4. "Deployed ✓"             │                               │
     │  ◄────────────────────────────────────────────────────────────│
```

No passwords stored anywhere. The token is short-lived and scoped to the specific action.

---

## Step 1: Create an Azure App Service

### Login to Azure

```bash
az login
```

### Create Resources

```bash
# 1. Create a resource group (logical container)
az group create \
  --name rg-react-demo \
  --location eastus2

# 2. Create an App Service plan (the VM that hosts your app)
#    F1 = free tier, --is-linux = Linux host
az appservice plan create \
  --name plan-react-demo \
  --resource-group rg-react-demo \
  --sku F1 \
  --is-linux

# 3. Create the web app
az webapp create \
  --name <CHOOSE-A-UNIQUE-NAME> \
  --resource-group rg-react-demo \
  --plan plan-react-demo \
  --runtime "NODE:22-lts"
```

> ⚠️ The web app name must be **globally unique** — it becomes `<name>.azurewebsites.net`. Try something like `react-app-<your-initials>`.

### Configure for Static Files

React builds to static HTML/CSS/JS. Configure App Service to serve them:

```bash
az webapp config set \
  --name <YOUR-APP-NAME> \
  --resource-group rg-react-demo \
  --startup-file "pm2 serve /home/site/wwwroot --no-daemon --spa"
```

The `--spa` flag tells pm2 to route all requests to `index.html` (required for client-side routing in React).

### Verify

```bash
# Should return the default page
curl https://<YOUR-APP-NAME>.azurewebsites.net
```

### Save These Values

You'll need them later:

| Value | Example | Where You'll Use It |
|-------|---------|---------------------|
| App name | `react-app-demo-srs` | Workflow file + GitHub variable |
| Resource group | `rg-react-demo` | Azure role assignment |
| Subscription ID | `dcb302da-...` | GitHub secret |

```bash
# Get your subscription ID
az account show --query id -o tsv
```

---

## Step 2: Create a GitHub Repository

```bash
# Navigate to your React project
cd /path/to/react-app-demo

# Initialize Git
git init
git add .
git commit -m "Initial commit: React app with Azure deployment"

# Create repo on GitHub and push
gh auth login                  # If not already logged in
gh repo create <your-username>/react-app-demo --public --source=. --push

# Ensure default branch is 'main'
git branch -M main
git push -u origin main
gh repo edit --default-branch main
```

### Verify

Visit `https://github.com/<your-username>/react-app-demo` — you should see your files.

---

## Step 3: Create an Azure AD App Registration

This creates an identity that GitHub Actions will use to authenticate with Azure.

```bash
# Create the app registration
az ad app create --display-name "github-react-deploy"
```

**Save these values from the output:**

```json
{
  "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",   ← Client ID (save this)
  "id": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"        ← Object ID (save this)
}
```

Create a service principal:

```bash
az ad sp create --id <APP_ID>
```

> 💡 **Tip:** If you already created an Azure AD app for another project (e.g., Foundry), you can reuse it — just add a new federated credential in Step 4.

---

## Step 4: Set Up OIDC Federation (Passwordless Auth)

This tells Azure: *"Trust tokens from GitHub Actions for this specific repository."*

### Create the Federated Credential

Create a file named `oidc-params.json`:

```json
{
  "name": "github-react-production",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<YOUR_GITHUB_USER>/react-app-demo:environment:production",
  "audiences": ["api://AzureADTokenExchange"]
}
```

Apply it:

```bash
# Get the app's Object ID
az ad app show --id <APP_ID> --query id -o tsv

# Create the federated credential
az ad app federated-credential create \
  --id <APP_OBJECT_ID> \
  --parameters @oidc-params.json
```

> ⚠️ **The `subject` must exactly match how your workflow authenticates.**
>
> | Workflow Setting | Required Subject |
> |------------------|------------------|
> | `environment: production` | `repo:<owner>/<repo>:environment:production` |
> | No environment, push to `main` | `repo:<owner>/<repo>:ref:refs/heads/main` |
>
> Our workflow uses `environment: production`, so we use the first format.

### Verify

```bash
az ad app federated-credential list \
  --id <APP_OBJECT_ID> \
  --query "[].{name:name, subject:subject}" -o table
```

---

## Step 5: Assign Azure Permissions

The service principal needs permission to deploy to the App Service.

### Required Role

| Role | Scope | Purpose |
|------|-------|---------|
| **Contributor** | Resource group containing the App Service | Deploy web apps, manage App Service settings |

### Assign the Role

```bash
# Get your subscription ID
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Assign Contributor on the resource group
az role assignment create \
  --assignee <APP_ID> \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-react-demo"
```

### Verify

```bash
az role assignment list \
  --assignee <APP_ID> \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-react-demo" \
  --query "[].roleDefinitionName" -o tsv
```

Expected output: `Contributor`

> 💡 **Scope tip:** Assigning at the resource group level (not the entire subscription) follows the principle of least privilege. The service principal can only manage resources in `rg-react-demo`.

### Comparison with Foundry Agent Permissions

| Scenario | Required Role(s) |
|----------|------------------|
| **React → App Service** | `Contributor` on resource group |
| **Foundry Agent** | `Azure AI User` + `Cognitive Services Contributor` on Foundry resource |

App Service deployment is simpler — a single `Contributor` role covers everything.

---

## Step 6: Configure GitHub Secrets and Variables

### Secrets (Encrypted, masked in logs)

```bash
cd /path/to/react-app-demo

gh secret set AZURE_CLIENT_ID --body "<your-app-id>"
gh secret set AZURE_TENANT_ID --body "<your-tenant-id>"
gh secret set AZURE_SUBSCRIPTION_ID --body "<your-subscription-id>"
```

### Finding Your Values

```bash
# Tenant ID
az account show --query tenantId -o tsv

# Subscription ID
az account show --query id -o tsv

# Client ID — the appId from Step 3
```

### Using GitHub Web UI (Alternative)

1. Go to your repo → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret** for each:

| Secret Name | Value |
|---|---|
| `AZURE_CLIENT_ID` | Your Azure AD app's Client ID |
| `AZURE_TENANT_ID` | Your Azure tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID |

---

## Step 7: Create a GitHub Environment

1. Go to your repo → **Settings** → **Environments**
2. Click **New environment**
3. Name it exactly: `production`
4. (Optional) Add **Required reviewers** for an approval gate
5. Click **Save protection rules**

---

## Step 8: Understand the Pipeline

### The Workflow File

Located at `.github/workflows/deploy-react-app.yml`:

```yaml
name: Deploy React App

# ── WHEN does this run? ──────────────────────────
on:
  push:
    branches: [main]        # On every push to main
  pull_request:
    branches: [main]        # On PRs targeting main (build only)

# ── WHAT permissions does the runner need? ────────
permissions:
  id-token: write           # Required for OIDC authentication
  contents: read            # Required to checkout code

# ── GLOBAL VARIABLES ─────────────────────────────
env:
  NODE_VERSION: '22'
  AZURE_WEBAPP_NAME: react-app-demo-srs    # ← Change this to your app name

jobs:
  # ── JOB 1: BUILD ──────────────────────────────
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Upload build artifact
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@v4
        with:
          name: react-build
          path: dist/

  # ── JOB 2: DEPLOY ─────────────────────────────
  deploy:
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: react-build
          path: dist/

      - name: Login to Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          package: dist/
```

### GitHub Actions Used

The workflow uses **6 pre-built actions** — reusable components from the GitHub Marketplace:

| Action | Publisher | What It Does | Why It's Needed |
|---|---|---|---|
| `actions/checkout@v4` | GitHub | Clones your repo onto the runner VM | Without this, the VM has no code |
| `actions/setup-node@v4` | GitHub | Installs Node.js + caches npm packages | Need Node.js to build React |
| `actions/upload-artifact@v4` | GitHub | Saves `dist/` between jobs | Build and deploy run on **separate** VMs |
| `actions/download-artifact@v4` | GitHub | Retrieves `dist/` on the deploy VM | Gets the built files for deployment |
| `azure/login@v2` | Microsoft | Exchanges OIDC token for Azure credentials | Passwordless authentication to Azure |
| `azure/webapps-deploy@v3` | Microsoft | Deploys files to Azure App Service | Zip-deploys `dist/` to your web app |

### Pipeline Flow Diagram

```
Pull Request to main          Push to main
       │                           │
       ▼                           ▼
┌─── build ────────────┐   ┌─── build ────────────────────────┐
│  ✅ Checkout          │   │  ✅ Checkout                     │
│  ✅ Setup Node.js     │   │  ✅ Setup Node.js                │
│  ✅ npm ci            │   │  ✅ npm ci                       │
│  ✅ npm run build     │   │  ✅ npm run build                │
│  ❌ Skip artifact     │   │  ✅ Upload dist/ as artifact     │
│     upload            │   └──────────────┬───────────────────┘
└──────────────────────┘                   │
       │                                   ▼
       ▼                    ┌─── deploy ───────────────────────┐
   DONE (PR validated)     │  🔒 Wait for approval (optional) │
                            │  📥 Download dist/ artifact      │
                            │  🔐 Azure OIDC login             │
                            │  🚀 Deploy to App Service        │
                            └──────────────────────────────────┘
                                           │
                                           ▼
                            🌐 https://<app>.azurewebsites.net
```

### What Happens on Pull Requests vs Pushes

| Event | Build Job | Deploy Job | Purpose |
|-------|-----------|------------|---------|
| **Pull Request** | ✅ Runs (builds + tests) | ❌ Skipped | Validate code compiles before merging |
| **Push to main** | ✅ Runs (builds + uploads artifact) | ✅ Runs (deploys to Azure) | Full deploy to production |

### Key Commands Explained

| Command | What It Does |
|---------|--------------|
| `npm ci` | Clean install — deletes `node_modules/`, installs exact versions from `package-lock.json`. Faster and more reliable than `npm install` for CI. |
| `npm run build` | Runs Vite to compile React JSX → optimized static files in `dist/` (HTML, CSS, JS, images) |

---

## Step 9: Push Code and Trigger the Pipeline

### Make a Change and Deploy

```bash
# Edit any file (e.g., src/App.jsx)

# Commit and push
git add .
git commit -m "Update homepage text"
git push origin main
```

The pipeline triggers automatically.

### Watch It Run

**GitHub CLI:**

```bash
# Watch the run in real time
gh run watch

# Or list recent runs
gh run list
```

**Web UI:**

Go to `https://github.com/<your-username>/react-app-demo/actions` → click the latest run.

### Verify Deployment

Open your browser to: `https://<YOUR-APP-NAME>.azurewebsites.net`

Or via command line:

```bash
curl -s -o /dev/null -w "%{http_code}" https://<YOUR-APP-NAME>.azurewebsites.net
# Should return: 200
```

---

## Step 10: Monitor and Troubleshoot

### GitHub CLI Commands

```bash
# List recent workflow runs
gh run list

# View a specific run's details
gh run view <RUN_ID>

# View logs for failed jobs only
gh run view <RUN_ID> --log-failed

# Re-run only the failed jobs
gh run rerun <RUN_ID> --failed

# Re-run the entire workflow
gh run rerun <RUN_ID>
```

### Azure CLI Commands

```bash
# Check App Service status
az webapp show --name <APP_NAME> --resource-group rg-react-demo \
  --query "{state:state, url:defaultHostName}" -o table

# View App Service logs (live)
az webapp log tail --name <APP_NAME> --resource-group rg-react-demo

# Restart the app
az webapp restart --name <APP_NAME> --resource-group rg-react-demo
```

---

## Common Errors and Fixes

### Error 1: OIDC Login Fails — "No matching federated identity record"

```
AADSTS700213: No matching federated identity record found for presented
assertion subject 'repo:user/repo:environment:production'
```

**Cause:** The `subject` in your federated credential doesn't match the workflow.

**Fix:** Create a credential with the exact subject shown in the error message. If your workflow uses `environment: production`, the subject must be `repo:<owner>/<repo>:environment:production`.

---

### Error 2: Deploy Fails — "Couldn't find a resource with name..."

```
Error: Couldn't find a resource with name "react-app-demo-srs"
```

**Cause:** The `AZURE_WEBAPP_NAME` in the workflow doesn't match the actual App Service name, or the service principal doesn't have access.

**Fix:**

1. Verify the app exists: `az webapp list --query "[].name" -o tsv`
2. Verify the name matches `env.AZURE_WEBAPP_NAME` in the workflow file
3. Ensure the service principal has `Contributor` on the resource group

---

### Error 3: Build Fails — "npm ci" Error

```
npm ERR! `npm ci` can only install packages when your package-lock.json
is in sync with package.json
```

**Cause:** `package-lock.json` is out of sync or missing.

**Fix:**

```bash
# Locally, regenerate the lock file
rm -rf node_modules package-lock.json
npm install
git add package-lock.json
git commit -m "Regenerate package-lock.json"
git push
```

---

### Error 4: App Loads but Shows "Cannot GET /path"

**Cause:** React client-side routing needs all requests to serve `index.html`, but the server is trying to find literal files.

**Fix:** Ensure the startup command has `--spa`:

```bash
az webapp config set \
  --name <APP_NAME> \
  --resource-group rg-react-demo \
  --startup-file "pm2 serve /home/site/wwwroot --no-daemon --spa"
```

---

### Error 5: Workflow Doesn't Trigger

**Possible causes:**

- You pushed to a branch other than `main`
- The workflow YAML has a syntax error
- The workflow file isn't at `.github/workflows/` (exact path required)

**Fix:** Verify the file is at `.github/workflows/deploy-react-app.yml` and push to `main`.

---

### Error 6: "Permission denied" on Azure Deploy

**Cause:** The service principal lacks the `Contributor` role.

**Fix:**

```bash
az role assignment create \
  --assignee <APP_ID> \
  --role "Contributor" \
  --scope "/subscriptions/<SUB_ID>/resourceGroups/rg-react-demo"
```

Wait 1–5 minutes for propagation, then re-run: `gh run rerun <RUN_ID> --failed`

---

## Security Best Practices

1. **Use OIDC, not stored credentials** — No passwords, no API keys, no rotation needed
2. **Use GitHub Secrets for Azure IDs** — They're encrypted and masked in logs
3. **Scope permissions narrowly** — Assign `Contributor` to the resource group, not the entire subscription
4. **Use environments with approval gates** — Require manual approval before deploying to production
5. **Use `npm ci` instead of `npm install`** — Ensures reproducible builds from `package-lock.json`
6. **Build and deploy are separate jobs** — The build VM never has Azure credentials

---

## Appendix: GitHub Actions Explained

### What Are GitHub Actions?

GitHub Actions are **pre-built, reusable automation components** published on the [GitHub Marketplace](https://github.com/marketplace?type=actions). Think of them as plugins.

Instead of writing complex shell scripts, you use actions:

```yaml
# Without an action (manual):
- run: |
    curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
    sudo apt-get install -y nodejs
    node --version

# With an action (simple):
- uses: actions/setup-node@v4
  with:
    node-version: '22'
```

### Anatomy of an Action Reference

```
uses: actions/checkout@v4
       ───────  ────────  ──
       │        │          │
       │        │          └─ Version tag (v4 = latest v4.x.x)
       │        └──────────── Action name
       └───────────────────── Publisher (actions = GitHub official)
```

### Actions Used in This Pipeline

#### `actions/checkout@v4`

- **Publisher:** GitHub (official)
- **Purpose:** Clones your repository onto the runner VM
- **Why needed:** Runner VMs start empty — without this, there's no code to build
- **Docs:** [github.com/actions/checkout](https://github.com/actions/checkout)

#### `actions/setup-node@v4`

- **Publisher:** GitHub (official)
- **Purpose:** Installs a specific Node.js version and caches npm packages
- **Why needed:** Runner VMs have Node.js pre-installed but may not have the version you need. Caching makes subsequent runs faster.
- **Key parameter:** `cache: 'npm'` — caches `node_modules/` across runs
- **Docs:** [github.com/actions/setup-node](https://github.com/actions/setup-node)

#### `actions/upload-artifact@v4` / `actions/download-artifact@v4`

- **Publisher:** GitHub (official)
- **Purpose:** Pass files between jobs (each job runs on a separate VM)
- **Why needed:** The build job creates `dist/`, but the deploy job runs on a different VM and needs those files
- **Retention:** Artifacts are kept for 90 days by default
- **Docs:** [github.com/actions/upload-artifact](https://github.com/actions/upload-artifact)

#### `azure/login@v2`

- **Publisher:** Microsoft
- **Purpose:** Authenticates with Azure using OIDC (or service principal credentials)
- **Why needed:** Subsequent Azure actions need an authenticated session
- **Key feature:** When using OIDC, no secrets are exchanged — GitHub's identity token is trusted directly
- **Docs:** [github.com/Azure/login](https://github.com/Azure/login)

#### `azure/webapps-deploy@v3`

- **Publisher:** Microsoft
- **Purpose:** Deploys files to Azure App Service via zip deploy
- **Why needed:** This is the actual deployment step — it packages `dist/` into a zip, uploads it to App Service, and triggers a restart
- **How it works:** Calls the App Service Kudu API to deploy the zip package
- **Docs:** [github.com/Azure/webapps-deploy](https://github.com/Azure/webapps-deploy)

---

## Quick Reference Card

### Setup Commands

| Step | Command |
|------|---------|
| Create resource group | `az group create --name rg-react-demo --location eastus2` |
| Create App Service plan | `az appservice plan create --name plan-react-demo --resource-group rg-react-demo --sku F1 --is-linux` |
| Create web app | `az webapp create --name <NAME> --resource-group rg-react-demo --plan plan-react-demo --runtime "NODE:22-lts"` |
| Configure SPA serving | `az webapp config set --name <NAME> --resource-group rg-react-demo --startup-file "pm2 serve /home/site/wwwroot --no-daemon --spa"` |
| Create AD app | `az ad app create --display-name "github-react-deploy"` |
| Create service principal | `az ad sp create --id <APP_ID>` |
| Create OIDC credential | `az ad app federated-credential create --id <OBJ_ID> --parameters @oidc-params.json` |
| Assign Contributor role | `az role assignment create --assignee <APP_ID> --role "Contributor" --scope <RG_SCOPE>` |

### GitHub CLI Commands

| Task | Command |
|------|---------|
| Set secret | `gh secret set <NAME> --body "<VALUE>"` |
| Set variable | `gh variable set <NAME> --body "<VALUE>"` |
| Watch run live | `gh run watch` |
| View failed logs | `gh run view <RUN_ID> --log-failed` |
| Re-run failed jobs | `gh run rerun <RUN_ID> --failed` |

### Required Permissions Summary

| Where | What | Why |
|-------|------|-----|
| **Azure AD** | App Registration + Service Principal | Identity for GitHub to authenticate as |
| **Azure AD** | Federated Credential (OIDC) | Trust GitHub tokens without passwords |
| **Azure** | `Contributor` role on resource group | Deploy to App Service |
| **GitHub** | Secrets: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` | Authentication parameters |
| **GitHub** | Environment: `production` | Approval gate for deployments |

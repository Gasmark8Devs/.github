# GitHub Actions Plugin Deployment Guide

> **Audience:** Developers, DevOps engineers, and engineering leads who need to understand, replicate, or maintain the automated plugin deployment process.
>
> **Purpose:** This is the detailed technical reference for plugin deployment automation in the `gasmark8` organization. The organization front page (`profile/README.md`) stays concise and links here for full implementation details.

## Related Repository Docs

- Organization front page (high-level): `profile/README.md`
- Reusable deployment workflow (source): `workflow-templates/plugin-deploy.yml`
- This implementation guide (detailed reference): `docs/github-actions-plugin-deployment.md`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Deployment Triggers](#2-deployment-triggers)
3. [Repository & Branch Strategy](#3-repository--branch-strategy)
4. [Workflow File Structure](#4-workflow-file-structure)
5. [Step-by-Step Workflow Configuration](#5-step-by-step-workflow-configuration)
6. [Secrets & Environment Setup](#6-secrets--environment-setup)
7. [Environment Promotion (Staging → Production)](#7-environment-promotion-staging--production)
8. [Monitoring & Notifications](#8-monitoring--notifications)
9. [Troubleshooting](#9-troubleshooting)
10. [Replicating This Setup in a New Repository](#10-replicating-this-setup-in-a-new-repository)
11. [Glossary](#11-glossary)

---

## 1. Overview

### What Is This Workflow?

This GitHub Actions workflow automates the full plugin deployment lifecycle — from code commit to production release — without requiring manual shell access or ad-hoc scripts. Every time a qualifying code change is pushed, the workflow:

1. Checks out the latest code.
2. Installs dependencies and builds the plugin artifact.
3. Runs automated tests to validate the build.
4. Packages the plugin (e.g., `.zip`, `.tar.gz`, or platform-specific bundle).
5. Deploys to the target environment (staging or production).
6. Sends a notification with the deployment result.

### Why GitHub Actions?

| Benefit | Details |
|---|---|
| **Native integration** | Runs directly inside GitHub — no third-party CI server to maintain. |
| **Audit trail** | Every run is logged with the triggering commit, actor, and timestamps. |
| **Reusability** | Shared workflows (stored here in `workflow-templates/`) can be called by every plugin repository in the org. |
| **Secrets management** | Credentials are stored as encrypted GitHub Secrets, never in source code. |

---

## 2. Deployment Triggers

The workflow supports three trigger modes. Choose the one that matches your intent.

### 2.1 Push to a Protected Branch

Automatically deploys to **staging** whenever code is merged into the `main` (or `develop`) branch.

```yaml
on:
  push:
    branches:
      - main        # triggers staging deployment
      - develop     # triggers dev/integration deployment
```

**When to use:** Day-to-day feature merges and hotfixes that should be validated in staging immediately.

---

### 2.2 Semantic Version Tag

Automatically deploys to **production** when a version tag matching `v*.*.*` is pushed.

```yaml
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'   # e.g., v1.4.2
```

**When to use:** Formal releases. Create and push the tag after staging validation is complete.

```bash
# Example: tag and push a production release
git tag v1.4.2
git push origin v1.4.2
```

---

### 2.3 Manual (Workflow Dispatch)

Allows any authorized team member to trigger a deployment from the GitHub UI without pushing code.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      plugin_version:
        description: 'Version to deploy (e.g., 1.4.2)'
        required: false
```

**When to use:** Re-deploying a specific version, emergency rollbacks, or deploying to production outside of the normal tag workflow.

**How to trigger manually:**
1. Open the repository on GitHub.
2. Click **Actions** → select the workflow name.
3. Click **Run workflow** (top-right).
4. Fill in the inputs and click **Run workflow**.

---

## 3. Repository & Branch Strategy

```
main  ──────────────────────────────────►  staging auto-deploy
  └── feature/xyz  (PR → merge to main)
  └── hotfix/abc   (PR → merge to main)

tag v*.*.*  ──────────────────────────►  production deploy
```

| Branch / Ref | Auto-Deploy Target | Requires Approval |
|---|---|---|
| `develop` | Dev / Integration | No |
| `main` | Staging | No |
| `v*.*.*` tag | Production | Yes (via GitHub Environment) |
| Manual dispatch | Configurable | Depends on environment |

---

## 4. Workflow File Structure

All shared workflows live in this `.github` repository under the `workflow-templates/` folder and can be referenced by plugin repositories via `uses:`.

```
gasmark8/.github
├── workflow-templates/
│   └── plugin-deploy.yml        ← reusable workflow definition
└── docs/
    └── github-actions-plugin-deployment.md  ← this document
```

Individual plugin repositories reference the shared workflow:

```
my-plugin-repo/
└── .github/
    └── workflows/
        └── deploy.yml           ← calls the shared workflow
```

---

## 5. Step-by-Step Workflow Configuration

### 5.1 Reusable Workflow (`workflow-templates/plugin-deploy.yml`)

This is the central definition. Plugin repos call this workflow and pass in their specific inputs.

```yaml
name: Plugin Deploy (Reusable)

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string           # 'staging' or 'production'
      plugin_name:
        required: true
        type: string
      plugin_version:
        required: true
        type: string
    secrets:
      DEPLOY_SSH_KEY:
        required: true
      DEPLOY_HOST:
        required: true
      DEPLOY_USER:
        required: true
      SLACK_WEBHOOK_URL:
        required: false

jobs:
  build-and-deploy:
    name: Build & Deploy – ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}   # links to GitHub Environment for approvals

    steps:
      # ── 1. Checkout ──────────────────────────────────────────────
      - name: Checkout code
        uses: actions/checkout@v4

      # ── 2. Set up Node.js (adjust version as needed) ─────────────
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      # ── 3. Install dependencies ───────────────────────────────────
      - name: Install dependencies
        run: npm ci

      # ── 4. Run tests ──────────────────────────────────────────────
      - name: Run tests
        run: npm test

      # ── 5. Build plugin artifact ──────────────────────────────────
      - name: Build plugin
        run: npm run build
        env:
          NODE_ENV: production

      # ── 6. Package artifact ───────────────────────────────────────
      - name: Package plugin
        run: |
          zip -r ${{ inputs.plugin_name }}-${{ inputs.plugin_version }}.zip dist/

      # ── 7. Upload artifact (for audit / rollback) ─────────────────
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ inputs.plugin_name }}-${{ inputs.plugin_version }}
          path: ${{ inputs.plugin_name }}-${{ inputs.plugin_version }}.zip
          retention-days: 30

      # ── 8a. Copy artifact to server ───────────────────────────────
      - name: Copy artifact to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          source: "${{ inputs.plugin_name }}-${{ inputs.plugin_version }}.zip"
          target: /tmp/

      # ── 8b. Extract and restart ────────────────────────────────────
      - name: Deploy to server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            DEPLOY_DIR="/var/www/plugins/${{ inputs.plugin_name }}"
            ARTIFACT="${{ inputs.plugin_name }}-${{ inputs.plugin_version }}.zip"
            mkdir -p "$DEPLOY_DIR"
            unzip -o "/tmp/$ARTIFACT" -d "$DEPLOY_DIR"
            systemctl restart "plugin-${{ inputs.plugin_name }}" || true

      # ── 9. Notify Slack ───────────────────────────────────────────
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && '✅' || '❌' }} *${{ inputs.plugin_name }} v${{ inputs.plugin_version }}* deployed to *${{ inputs.environment }}* by ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

### 5.2 Caller Workflow in a Plugin Repository (`.github/workflows/deploy.yml`)

Each plugin repository contains a thin caller workflow that passes its specific values into the shared reusable workflow above.

```yaml
name: Deploy Plugin

on:
  push:
    branches:
      - main
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  # ── Staging: triggered by push to main ──────────────────────────
  deploy-staging:
    if: github.ref == 'refs/heads/main' || (github.event_name == 'workflow_dispatch' && inputs.environment == 'staging')
    uses: gasmark8/.github/workflow-templates/plugin-deploy.yml@main
    with:
      environment: staging
      plugin_name: my-awesome-plugin
      plugin_version: ${{ github.sha }}
    secrets:
      DEPLOY_SSH_KEY: ${{ secrets.STAGING_DEPLOY_SSH_KEY }}
      DEPLOY_HOST: ${{ secrets.STAGING_DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.STAGING_DEPLOY_USER }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # ── Production: triggered by version tag ────────────────────────
  deploy-production:
    if: startsWith(github.ref, 'refs/tags/v') || (github.event_name == 'workflow_dispatch' && inputs.environment == 'production')
    uses: gasmark8/.github/workflow-templates/plugin-deploy.yml@main
    with:
      environment: production
      plugin_name: my-awesome-plugin
      plugin_version: ${{ github.ref_name }}
    secrets:
      DEPLOY_SSH_KEY: ${{ secrets.PROD_DEPLOY_SSH_KEY }}
      DEPLOY_HOST: ${{ secrets.PROD_DEPLOY_HOST }}
      DEPLOY_USER: ${{ secrets.PROD_DEPLOY_USER }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 6. Secrets & Environment Setup

### 6.1 What Are GitHub Secrets?

GitHub Secrets are encrypted key-value pairs stored at the repository or organization level. They are injected into workflow runs as environment variables and are **never** printed in logs.

### 6.2 Required Secrets

Configure these in **Settings → Secrets and variables → Actions** for each repository (or at the organization level for shared access).

| Secret Name | Scope | Description |
|---|---|---|
| `STAGING_DEPLOY_SSH_KEY` | Repository | Private SSH key for staging server access |
| `STAGING_DEPLOY_HOST` | Repository | Hostname or IP of the staging server |
| `STAGING_DEPLOY_USER` | Repository | SSH username on the staging server |
| `PROD_DEPLOY_SSH_KEY` | Repository | Private SSH key for production server access |
| `PROD_DEPLOY_HOST` | Repository | Hostname or IP of the production server |
| `PROD_DEPLOY_USER` | Repository | SSH username on the production server |
| `SLACK_WEBHOOK_URL` | Org or Repository | Slack Incoming Webhook URL for notifications |

> **Security tip:** Never commit private keys or passwords to the repository. Always use GitHub Secrets.

### 6.3 How to Add a Secret

1. Go to the repository on GitHub.
2. Click **Settings** → **Secrets and variables** → **Actions**.
3. Click **New repository secret**.
4. Enter the **Name** (exactly as shown in the table above) and the **Value**.
5. Click **Add secret**.

For organization-wide secrets:

1. Go to **Organization Settings** → **Secrets and variables** → **Actions**.
2. Click **New organization secret**.
3. Set the **Repository access** policy (e.g., all repositories, or selected ones).

### 6.4 GitHub Environments

GitHub Environments add an extra layer of control for production deployments:

1. Go to **Settings** → **Environments** → **New environment**.
2. Name it `production`.
3. Enable **Required reviewers** and add the team leads who must approve production deploys.
4. Optionally set a **Wait timer** (e.g., 5 minutes) before the deployment proceeds.

```
Developer pushes tag v1.4.2
         │
         ▼
 GitHub Actions starts
         │
         ▼
  Environment: production
  ⏳ Waiting for approval from: @alfcastillo90
         │
    (Approved)
         │
         ▼
   Deployment runs
```

---

## 7. Environment Promotion (Staging → Production)

The recommended release flow:

```
1. Merge feature branch → main
        ↓
2. GitHub Actions auto-deploys to staging
        ↓
3. QA / director validates on staging
        ↓
4. Tag the commit: git tag v1.4.2 && git push origin v1.4.2
        ↓
5. GitHub Actions triggers production deployment
        ↓
6. Required reviewer approves in GitHub UI
        ↓
7. Plugin is live in production
        ↓
8. Slack notification confirms success (or failure)
```

---

## 8. Monitoring & Notifications

### Viewing Run History

1. Open the repository on GitHub.
2. Click **Actions** tab.
3. Select the workflow from the left sidebar.
4. Click any run to see logs, timing, and artifact downloads.

### Slack Notifications

Each deployment (success or failure) posts a message to the configured Slack channel:

- ✅ `my-awesome-plugin v1.4.2 deployed to production by alfcastillo90`
- ❌ `my-awesome-plugin v1.4.2 deployment to production FAILED – see GitHub Actions for details`

### Email Notifications (Default GitHub Behavior)

GitHub automatically emails the commit author when a workflow they triggered fails. This can be configured in **Profile → Notifications**.

---

## 9. Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Workflow does not trigger on push | Branch name mismatch in `on.push.branches` | Verify the branch name in the workflow file matches exactly. |
| `Permission denied (publickey)` during SSH deploy | Wrong or missing `DEPLOY_SSH_KEY` secret | Re-generate an SSH key pair, update the secret, and ensure the public key is in `~/.ssh/authorized_keys` on the server. |
| Deployment succeeds but plugin not updated | Incorrect remote path in `script:` block | Check the `cd` path in the SSH deploy step. |
| Production deployment stuck at "Waiting for approval" | No reviewer has approved | Open the workflow run → click **Review deployments** → approve. |
| `npm ci` fails with missing packages | `package-lock.json` out of sync | Run `npm install` locally, commit the updated `package-lock.json`, and re-push. |
| Artifact not found after 30 days | Default retention expired | Increase `retention-days` in the `upload-artifact` step, or download the artifact before it expires. |

---

## 10. Replicating This Setup in a New Repository

Follow these steps to add the deployment workflow to any new plugin repository in the organization:

### Step 1 — Create the caller workflow file

In the new repository, create `.github/workflows/deploy.yml` and paste the [caller workflow from Section 5.2](#52-caller-workflow-in-a-plugin-repository-githubworkflowsdeployyml), updating `plugin_name` to match your plugin.

### Step 2 — Add required secrets

Add all secrets listed in [Section 6.2](#62-required-secrets) under **Settings → Secrets and variables → Actions**.

### Step 3 — Set up GitHub Environments

Create `staging` and `production` environments as described in [Section 6.4](#64-github-environments). Add required reviewers for `production`.

### Step 4 — Ensure SSH access

On your staging and production servers, add the **public key** that corresponds to `DEPLOY_SSH_KEY` to the `~/.ssh/authorized_keys` file for `DEPLOY_USER`.

```bash
# On the server, run:
echo "ssh-ed25519 AAAA... your-key-comment" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Step 5 — Verify with a test push

Merge a small change to `main` and confirm the staging deployment completes successfully in the Actions tab.

### Step 6 — Tag a production release

```bash
git tag v0.1.0
git push origin v0.1.0
```

Approve the deployment in GitHub and confirm the plugin is live.

---

## 11. Glossary

| Term | Definition |
|---|---|
| **GitHub Actions** | GitHub's built-in CI/CD automation platform. Workflows are defined in YAML files inside `.github/workflows/`. |
| **Workflow** | A YAML file that defines one or more automated jobs triggered by events (push, tag, manual, etc.). |
| **Job** | A unit of work inside a workflow. Jobs run in parallel by default unless dependencies are declared. |
| **Step** | An individual task within a job — either a shell command (`run:`) or a reusable action (`uses:`). |
| **Action** | A pre-built, shareable step. Examples: `actions/checkout`, `actions/setup-node`. |
| **Reusable Workflow** | A workflow that can be called by other workflows using `workflow_call`. Stored centrally to reduce duplication. |
| **Secret** | An encrypted variable stored in GitHub, injected into workflows at runtime. Never visible in logs. |
| **Environment** | A named deployment target (e.g., `staging`, `production`) with optional approval gates and protection rules. |
| **Artifact** | A file or folder produced by a workflow run (e.g., the compiled plugin `.zip`). Can be downloaded for 30 days by default. |
| **Workflow Dispatch** | A manual trigger that lets authorized users start a workflow from the GitHub UI or API. |
| **SSH Key Pair** | A public/private cryptographic key pair. The private key is stored as a GitHub Secret; the public key is placed on the target server. |

---

*Last updated: April 2026 | Maintained by: gasmark8 engineering team*

# GitHub Actions Plugin Deployment Guide

> **Audience:** Developers, DevOps engineers, and engineering leads who need to understand, replicate, or maintain the automated plugin deployment process.
>
> **Purpose:** This is the detailed technical reference for plugin deployment automation in the `gasmark8` organization. The organization front page ([`profile/README.md`](../profile/README.md)) stays concise and links here for full implementation details.

## Related Repository Docs

- Organization front page (high-level): [`profile/README.md`](../profile/README.md)
- Copy-paste workflow target path: `.github/workflows/deploy.yml`
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
   - [7.1 Production Promotion Strategies](#71-production-promotion-strategies)
8. [Monitoring & Verification](#8-monitoring--verification)
9. [Troubleshooting](#9-troubleshooting)
10. [Replicating This Setup in a New Repository](#10-replicating-this-setup-in-a-new-repository)
11. [Glossary](#11-glossary)

---

## 1. Overview

### What Is This Workflow?

This guide focuses on the deployment pattern we use most in production: **WordPress plugin deployment via `rsync` over SSH password auth** from GitHub Actions.

The baseline workflow:

1. Is manually triggered from GitHub Actions (`workflow_dispatch`).
2. Lets the operator choose `staging` or `production`.
3. Uses environment-scoped secrets for server credentials.
4. Syncs plugin files directly to the server with `rsync`.
5. Optionally runs post-deploy PHP sync scripts on the server.

### Why GitHub Actions?

| Benefit | Details |
|---|---|
| **Native integration** | Runs directly inside GitHub — no third-party CI server to maintain. |
| **Audit trail** | Every run is logged with the triggering commit, actor, and timestamps. |
| **Reusability** | The same `deploy.yml` pattern can be copied across WordPress plugin repositories. |
| **Secrets management** | Credentials are stored as encrypted GitHub Secrets, never in source code. |

---

## 2. Deployment Triggers

This WordPress plugin workflow supports three trigger modes. For most plugin deployments, we recommend starting with **manual dispatch**.

### 2.1 Manual (Workflow Dispatch) - Recommended

Allows an authorized team member to deploy from the GitHub UI without pushing extra commits.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      confirm_production:
        description: 'Production only - check to confirm production deploy'
        required: false
        default: false
        type: boolean
```

**When to use:** Standard WordPress plugin deploys where operators intentionally choose the target environment.

---

### 2.2 Push to Branch (Optional)

Automatically deploys on merge to specific branches.

```yaml
on:
  push:
    branches:
      - main
      - develop
```

**When to use:** Teams that want automatic non-production deployment on merge.

---

### 2.3 Semantic Version Tag (Optional)

Automatically deploys when a semantic version tag is pushed.

```yaml
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
```

**When to use:** Release-driven workflows where production deploys are tied to tags.

**How to trigger manually (`workflow_dispatch`):**
1. Open the repository on GitHub.
2. Click **Actions** → select the workflow name.
3. Click **Run workflow** (top-right).
4. Fill in the inputs and click **Run workflow**.

---

## 3. Repository & Branch Strategy

The baseline WordPress deployment pattern in this guide is **manual dispatch** with environment selection.
If your team also uses branch and tag automation, this is the usual model:

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

For this pattern, each plugin repository contains its own deploy workflow file.
The same file contents can be reused across repositories with minimal edits.

```
my-plugin-repo/
└── .github/
    └── workflows/
        └── deploy.yml           ← workflow from Section 5.1
```

---

## 5. Step-by-Step Workflow Configuration

### 5.1 Copy-Paste Workflow for WordPress Plugins (`.github/workflows/deploy.yml`)

This is the reusable deployment pattern used in multiple WordPress plugin repositories.
It deploys via `rsync` over SSH password auth using environment-scoped secrets.
It is based on the same structure currently used in `gm8gmcastats/.github/workflows/deploy.yml`.

Core deployment values used by this pattern:

- `DEPLOY_HOST` (secret)
- `DEPLOY_USER` (secret)
- `DEPLOY_PATH` (workflow variable)
- `DEPLOY_PASSWORD` (secret)

```yaml
name: Deploy plugin

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      confirm_production:
        description: 'Production only - check to confirm you intend to deploy to production'
        required: false
        default: false
        type: boolean
      run_sync_scripts:
        description: 'Run php scripts/ (sync) on server'
        required: false
        default: false
        type: boolean

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    env:
      SSHPASS: ${{ secrets.DEPLOY_PASSWORD }}
      DEPLOY_PATH: '~/public_html/wp-content/plugins/your-plugin-slug'

    steps:
      - name: Block production without confirmation
        if: ${{ inputs.environment == 'production' }}
        run: |
          case "${{ inputs.confirm_production }}" in
            true|True) exit 0 ;;
            *) echo "::error::Production deploy blocked: check Confirm production in the workflow form, then run again."; exit 1 ;;
          esac

      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'

      - name: Install Composer dependencies
        run: composer install --no-dev --prefer-dist --no-interaction

      - name: Install sshpass
        run: sudo apt-get update -qq && sudo apt-get install -y sshpass

      - name: Deploy via rsync (SSH password)
        run: |
          rsync -avz --delete \
            -e "sshpass -e ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null" \
            --exclude '.git' \
            --exclude '.github' \
            --exclude '.gitignore' \
            --exclude '.env' \
            --exclude '.env.*' \
            --exclude '*.log' \
            --exclude '.claude' \
            --exclude 'docs/' \
            ./ "${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}:${{ env.DEPLOY_PATH }}/"

      - name: Run plugin sync scripts on server (optional)
        if: ${{ inputs.run_sync_scripts }}
        run: |
          sshpass -e ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
            "${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}" \
            "cd ${{ env.DEPLOY_PATH }} && php scripts/sync_eportfolio_data.php && php scripts/sync_activities_detailed.php"
```

### 5.2 What to Customize Per Plugin

Before using the workflow above, update:

1. `DEPLOY_PATH` to your plugin location (for example: `~/public_html/wp-content/plugins/my-plugin`).
2. Optional script names in the last SSH step (or remove the step entirely).
3. PHP version in `Setup PHP` if your server stack is different.
4. Optional `rsync --exclude` patterns for plugin-specific files.

---

## 6. Secrets & Environment Setup

### 6.1 What Are GitHub Secrets?

GitHub Secrets are encrypted key-value pairs stored at the repository or organization level. They are injected into workflow runs as environment variables and are **never** printed in logs.

### 6.2 Required Secrets

Configure these in **Settings → Secrets and variables → Actions** inside each GitHub Environment (`staging` and `production`).
Use the same secret names in both environments, with different values.

| Secret Name | Scope | Description |
|---|---|---|
| `DEPLOY_HOST` | Environment (`staging`/`production`) | Hostname or IP of the destination server |
| `DEPLOY_USER` | Environment (`staging`/`production`) | SSH username used for deployment |
| `DEPLOY_PASSWORD` | Environment (`staging`/`production`) | SSH password used by `sshpass` |

### 6.3 Runtime Variables (Non-Secret)

These are configured in the workflow file itself:

| Variable | Where | Description |
|---|---|---|
| `DEPLOY_PATH` | `jobs.deploy.env` | Target plugin directory on the remote WordPress server |

> **Security tip:** Never commit private keys or passwords to the repository. Always use GitHub Secrets.

### 6.4 How to Add a Secret

1. Go to the repository on GitHub.
2. Click **Settings** → **Secrets and variables** → **Actions**.
3. Open environment **staging**.
4. Click **Add environment secret** and create:
   - `DEPLOY_HOST`
   - `DEPLOY_USER`
   - `DEPLOY_PASSWORD`
5. Repeat in environment **production** with production values.

### 6.5 GitHub Environments

GitHub Environments add an extra layer of control for production deployments:

1. Go to **Settings** → **Environments** → **New environment**.
2. Create both `staging` and `production`.
3. Add `DEPLOY_HOST`, `DEPLOY_USER`, and `DEPLOY_PASSWORD` in each environment.
4. In `production`, enable **Required reviewers** (recommended).
5. Optionally set a **Wait timer** (e.g., 5 minutes) before deployment proceeds.

```
Developer runs workflow_dispatch
         │
         ▼
 GitHub Actions starts
         │
         ▼
  Environment: production
  ⏳ Waiting for required reviewer approval
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
1. Open GitHub Actions → Deploy plugin
        ↓
2. Run manually with environment = staging
        ↓
3. QA / director validates on staging
        ↓
4. Run manually again with environment = production
        ↓
5. Required reviewer approves in GitHub UI
        ↓
6. Plugin is live in production
```

### 7.1 Production Promotion Strategies

There is no single "best" strategy for every team. These are the common production promotion approaches:

| Strategy | How it works | Strengths | Tradeoffs | Best fit |
|---|---|---|---|---|
| Branch-based release | Deploy on push/merge to a branch such as `main` or `release/*`. | Fast and simple; easy for teams already shipping from branches. | Higher risk of accidental production deploys without extra safeguards. | Teams with strict PR controls and mature branch discipline. |
| Tag-based release | Deploy only when pushing a semantic tag like `v1.4.2`. | Strong traceability; immutable release point; easy rollback by redeploying older tags. | Requires release/tag process discipline. | Teams that want clear versioned production history. |
| GitHub Release-based | Deploy when a GitHub Release is published (often tied to a tag). | Combines deployment trigger with release notes and stakeholder visibility. | One extra release step to manage. | Teams that treat release notes as part of release governance. |
| Manual dispatch | Operator runs `workflow_dispatch` and selects `production`. | Maximum operational control and flexibility. | Human-dependent process; weaker release traceability if not paired with conventions. | Small teams and ops-driven deploys (your current baseline). |
| Artifact promotion | Build once (staging), then promote the exact same artifact to production. | Highest consistency between tested and released output. | More pipeline design overhead (artifact storage and promotion logic). | Larger teams needing strict release reproducibility. |

#### Copy-paste trigger examples

**A) Branch-based production trigger**

```yaml
on:
  push:
    branches:
      - main
```

**B) Tag-based production trigger**

```yaml
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
```

**C) GitHub Release-based production trigger**

```yaml
on:
  release:
    types: [published]
```

**D) Manual production trigger**

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
```

#### Recommended decision model

- If you want speed and control: use **manual dispatch** + required reviewers.
- If you want release traceability: use **tag-based** promotion.
- If you want communication/governance around releases: use **GitHub Release-based** promotion.
- If you want strict reproducibility at scale: use **artifact promotion**.
- If you already ship from branch and it works: keep **branch-based**, but protect it with required reviewers and branch protection rules.

---

## 8. Monitoring & Verification

### Viewing Run History

1. Open the repository on GitHub.
2. Click **Actions** tab.
3. Select the workflow from the left sidebar.
4. Click any run to see logs, timing, and step outcomes.

### Post-Deploy Verification Checklist

After each deployment, verify:

1. The workflow run status is green.
2. Files exist on the target server plugin path.
3. The plugin is active and functional in WordPress admin.
4. Optional sync scripts completed without errors (if enabled).

---

## 9. Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Workflow does not appear in Actions | Wrong file path or syntax error in YAML | Confirm file is at `.github/workflows/deploy.yml` and validate YAML format. |
| `Permission denied` during deploy | Wrong `DEPLOY_USER`/`DEPLOY_PASSWORD`, or SSH is blocked | Verify credentials in environment secrets and confirm SSH access from outside GitHub Actions. |
| Deployment succeeds but plugin is not updated | Wrong `DEPLOY_PATH` | Validate `DEPLOY_PATH` points to `wp-content/plugins/<plugin-slug>`. |
| Host key warning / SSH prompt interrupts command | Strict host checking prompts in non-interactive CI | Keep non-interactive SSH flags exactly as shown in the workflow. |
| Production deployment stuck at "Waiting for approval" | No reviewer has approved | Open the workflow run → click **Review deployments** → approve. |
| Optional sync scripts fail | Wrong script name/path or missing runtime dependency | SSH to the server and run each script manually to confirm path and PHP compatibility. |

---

## 10. Replicating This Setup in a New Repository

Follow these steps to add this workflow to any new WordPress plugin repository in the organization:

### Step 1 — Create the workflow file

In the new repository, create `.github/workflows/deploy.yml` and paste the copy-paste workflow from [Section 5.1](#51-copy-paste-workflow-for-wordpress-plugins-githubworkflowsdeployyml).

### Step 2 — Add required secrets

Add `DEPLOY_HOST`, `DEPLOY_USER`, and `DEPLOY_PASSWORD` for both environments as shown in [Section 6.2](#62-required-secrets).

### Step 3 — Set up GitHub Environments

Create `staging` and `production` environments as described in [Section 6.5](#65-github-environments). Add required reviewers for `production`.

### Step 4 — Configure plugin path in workflow

Set `DEPLOY_PATH` in `jobs.deploy.env` to the plugin directory on each server (for example: `~/public_html/wp-content/plugins/my-plugin`).

### Step 5 — Verify with a staging run

Run the workflow with `environment = staging` and confirm the plugin updates correctly.

### Step 6 — Promote to production

Run the workflow again with `environment = production`, check **Confirm production**, approve deployment, and validate the plugin in production.

---

## 11. Glossary

| Term | Definition |
|---|---|
| **GitHub Actions** | GitHub's built-in CI/CD automation platform. Workflows are defined in YAML files inside `.github/workflows/`. |
| **Workflow** | A YAML file that defines one or more automated jobs triggered by events (push, tag, manual, etc.). |
| **Job** | A unit of work inside a workflow. Jobs run in parallel by default unless dependencies are declared. |
| **Step** | An individual task within a job — either a shell command (`run:`) or a reusable action (`uses:`). |
| **Action** | A pre-built, shareable step. Examples: `actions/checkout`, `actions/setup-node`. |
| **Environment Secret** | A secret value scoped to a GitHub Environment (for example, `staging` or `production`). |
| **Secret** | An encrypted variable stored in GitHub, injected into workflows at runtime. Never visible in logs. |
| **Environment** | A named deployment target (e.g., `staging`, `production`) with optional approval gates and protection rules. |
| **`rsync`** | A command-line utility used here to synchronize plugin files from the repository runner to the remote server. |
| **Workflow Dispatch** | A manual trigger that lets authorized users start a workflow from the GitHub UI or API. |
| **`sshpass`** | A utility that provides SSH passwords non-interactively in CI using the `SSHPASS` environment variable. |

---

*Last updated: April 2026 | Maintained by: gasmark8 engineering team*

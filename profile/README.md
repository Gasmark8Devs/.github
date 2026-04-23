# Gasmark8

This `.github` repository powers the public organization profile for `gasmark8`.
It is the first documentation page visitors see on the organization home.

## What this repository does

- Centralizes shared CI/CD conventions across plugin repositories.
- Provides onboarding and operational references for engineering workflows.
- Serves as the main navigation entry point for core documentation.

## Quick navigation

- Full GitHub Actions deployment guide: [Plugin Deployment Guide](../docs/github-actions-plugin-deployment.md)
- Reusable deployment workflow source: [`workflow-templates/plugin-deploy.yml`](../workflow-templates/plugin-deploy.yml)

## Standard deployment flow (summary)

1. Merge to `main` to deploy to staging.
2. Validate changes in staging (QA/team).
3. Push a semantic tag (`v*.*.*`) to deploy to production.
4. Complete GitHub Environment approval when required.

## Scope of this page

This README is intentionally high-level.
For full implementation details (YAML examples, secrets, troubleshooting), use the [Plugin Deployment Guide](../docs/github-actions-plugin-deployment.md).

## Maintenance

If deployment conventions change, update:

1. `workflow-templates/plugin-deploy.yml`
2. `docs/github-actions-plugin-deployment.md`
3. `profile/README.md` (only if front-page messaging changes)

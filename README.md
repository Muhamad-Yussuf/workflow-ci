
A collection of reusable GitHub Actions workflows for container build/push, GCP Compute Engine (GCE) deployment, and environment updates using GCP Workload Identity Federation (OIDC), with no long-lived service account keys.

## 🔍 Overview
This repository provides centralized CI/CD concerns for  projects:
- Docker image build + Artifact Registry push (`docker-build-push.yml`)
- GCE deployment via IAP/OS Login (`deploy-gce-compose.yml`)
- VM env file sync from Secret Manager (`update-env.yml`)

The goal is to let calling repositories include these workflows with minimal duplication.

## 📦 Workflows
### 1. docker-build-push.yml
- Builds Docker image
- Uses build cache (optional)
- Pushes to Artifact Registry
- Typical call from repo workflow with inputs: image name, registry path, tags

### 2. deploy-gce-compose.yml
- Connects to target GCE VM through IAP tunnel
- Uses OS Login account credentials
- Updates environment variables and runs `docker compose up -d`
- Safe for zero-downtime rollouts when compose stacks are configured correctly

### 3. update-env.yml
- Fetches a Secret Manager secret value from GCP
- Writes to `.env` on the VM
- Performs a service restart (via `docker compose restart` or process command)

## 🔐 Required permissions and secrets
In calling repositories you must configure:
- `GCP_WIF_PROVIDER`: full workload identity provider resource path (`projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<POOL_ID>/providers/<PROVIDER_ID>`)
- `GCP_PROJECT_ID` (if workflows expect it)
- `GCP_GCE_INSTANCE` / `GCP_GCE_ZONE` / `GCP_GCE_PROJECT` as needed
- VM OS Login and IAP permissions for the GitHub identity
- Artifact Registry push permissions: `artifactregistry.repositories.uploadArtifacts`
- Secret Manager access: `secretmanager.versions.access`

## 🔐 Authentication approach
- This repository intentionally does NOT store or use long-lived Service Account keys in secrets.
- Workflows use Workload Identity Federation (WIF) via OIDC through `google-github-actions/auth@v2`.
- GitHub Actions obtains short-lived tokens at runtime, then impersonates a GCP service account (via `service_account` input) to reduce credential risk.
- If a per-env override is required, `wif_provider_dev`/`wif_provider_prod` are supported alongside repo-level `GCP_WIF_PROVIDER`.
- Recommended: do not use raw service account JSON ambient clouds because these workflows are designed for OIDC/WIF.


## ⚙️ Usage
1. In your target repository, create `.github/workflows/ci.yml`.
2. Call the reusable workflow:

```yaml
name: CI/CD
on:
  push:
    branches: [main]

jobs:
  build-and-push:
    uses: Muhamad-Yussuf/workflow-ci/.github/workflows/docker-build-push.yml@main
    with:
      image: us-central1-docker.pkg.dev/my-project/my-repo/my-image
      tags: ${{ github.sha }}

  deploy:
    needs: build-and-push
    uses: Muhamad-Yussuf/workflow-ci/.github/workflows/deploy-gce-compose.yml@main
    with:
      gce-instance: my-vm
      gce-zone: us-central1-a
      compose-file: docker-compose.yml

  update-env:
    uses: Muhamad-Yussuf/workflow-ci/.github/workflows/update-env.yml@main
    with:
      secret-name: projects/my-project/secrets/my-env-secret
      target-file: /home/runner/app/.env
```
```

## 🧩 Best practices
- Keep service account RBAC minimal and focused.
- Use dedicated workloads for GitHub actions identity.
- Test each workflow path in a staging project first.
- Pin reusable workflow versions (`@v1` or a specific commit SHA) when stable.

## 🛠️ Contributing
1. Fork the repository.
2. Add or improve workflow YAML and tests.
3. Open a PR with details and verification steps.

## 📝 Notes
- This repository assumes your VM has Docker and Docker Compose installed.
- For secure secrets with `update-env.yml`, store them in Secret Manager and approve access explicitly.
- `deploy-gce-compose.yml` uses an inline sed update and compose up; ensure your `compose_service` is correct for your stack.

## 🚨 Security & reliability checklist
- Ensure `GCP_WIF_PROVIDER`, service accounts and roles are provisioned with least privilege.
- Use short TTL for OS Login keys (current workflows use 5m).
- Use `gcloud compute os-login ssh-keys add` only from expected branches/environments.
- Avoid running these workflows against production until validated in staging.
- Pin dependency actions (e.g., `google-github-actions/auth@v2`) or provide commit hashes for enterprise guardrails.

## 🧭 Workflow adoption matrix
- `docker-build-push.yml`: use for container image CI with caching and non-PR promotion logic
- `deploy-gce-compose.yml`: use for containerized app rollouts on a single GCE instance via IAP + OS Login
- `update-env.yml`: use when per-customer `.env` from Secret Manager must be synced and service restarted
- `update-env-generic.yml`: use for simple `.env` sync plus optional post-deploy command across envs

## 📄 License
Specify your org license here (e.g., MIT, Apache-2.0).

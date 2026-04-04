---
name: workflow-infisical-aliyun
description: Audits GitHub Actions workflows for secrets/env usage, wires Infisical OIDC secret fetch (file + env exports), migrates container registry steps from Docker Hub/GHCR to Aliyun ACR, and adds post-release ECS deploy via reusable workflow. Use when editing CI/CD, Infisical integration, Aliyun ACR/ECS deployment, or replacing docker login/push targets.
---

# GitHub Workflow: Infisical Secrets + Aliyun ACR + ECS

## When to apply

- Changing `.github/workflows/*.yml` or reusable workflows.
- Replacing `secrets.*` / hardcoded registry URLs with centralized secrets.
- Moving image publish from GHCR/Docker Hub to Aliyun Container Registry (ACR).
- Adding ECS deploy after container images are tagged (for example after `post-release` sets `latest`).

## 1) Audit workflows for secrets and environment variables

1. Search for: `secrets.`, `env:`, `${{ secrets.`, `GITHUB_TOKEN`, `password:`, `token`, `registry`, `docker/login`, `DOCKER_`, `GHCR`, `ghcr.io`, `docker.io`.
2. Classify each value:
   - **Runtime app config** (consumed inside build/test containers or app): prefer Infisical **file** export (for example `.env` consumed by a step).
   - **CI/deploy orchestration** (registry URLs, kubectl/SSH, Terraform): prefer Infisical **env** export into the job `env` block or step `env`.
3. Do not commit real values. Map names to Infisical keys and document required keys in the workflow comment or team runbook.

## 2) Infisical fetch patterns (GitHub Actions)

Use two steps when both app bundle secrets and deploy-time variables are needed:

**Application secrets as a file** (`export-type: file` — path set by the action; consume in later steps):

```yaml
- name: Fetch secrets from Infisical (app file)
  uses: Infisical/secrets-action@v1.0.15
  with:
    method: oidc
    identity-id: ${{ secrets.INFISICAL_OIDC_IDENTITY_ID }}
    project-slug: <your-infisical-project-slug>
    env-slug: prod
    export-type: file
    secret-path: /${{ github.event.repository.name }}
```

**Deployment / CI variables as environment** (`export-type: env`):

```yaml
- name: Fetch secrets from Infisical (env)
  uses: Infisical/secrets-action@v1.0.15
  with:
    method: oidc
    identity-id: ${{ secrets.INFISICAL_OIDC_IDENTITY_ID }}
    project-slug: <your-infisical-project-slug>
    env-slug: prod
    export-type: env
    secret-path: /${{ github.event.repository.name }}
```

### `secret-path` convention

- Use the **repository short name only** (no `owner/` prefix), aligned with Infisical folder paths such as `/openclaw`.
- In workflows, set `secret-path: /${{ github.event.repository.name }}` so it resolves to `/<repo>` for the current repository.
- If multiple organizations could reuse the same short name inside one Infisical project, use separate Infisical projects or a different path scheme — the short-name default assumes names are unique enough for your layout.
- Parameterize `identity-id` and `project-slug` via `secrets` / `vars` — **do not hardcode** org-specific IDs in shared workflow templates.

### Permissions for OIDC

Jobs that use `method: oidc` need:

```yaml
permissions:
  id-token: write
  contents: read # plus whatever else the job needs
```

## 3) Bootstrap secrets in Infisical (CLI)

Goal: take variables discovered in step 1 and create/update them in Infisical without pasting values into git.

1. Install CLI locally or in a one-off authenticated session (see Infisical docs for `infisical login` / service token).
2. From a local `.env` or scanned list of `KEY=value` lines:
   - Prefer **`infisical secrets set`** (or project bulk import commands supported by your Infisical version) targeting the same `project`, `environment`, and path as `secret-path`.
3. Verify in the Infisical UI that the folder path matches the short repo name (for example `/openclaw` for repository `openclaw`).
4. Rotate any credentials that were ever committed; rely on Infisical as source of truth going forward.

(Exact subcommands change across CLI versions — confirm against current Infisical CLI help before scripting.)

## 4) Replace Docker Hub / GHCR with Aliyun ACR

After secrets exist in Infisical (exported to env in CI):

1. Define env (from Infisical `export-type: env` or `vars`):

```yaml
env:
  ACR_REGISTRY: registry.<region>.aliyuncs.com # example
  ACR_USERNAME: ${{ env.ACR_USERNAME }} # populated after Infisical fetch
  ACR_PASSWORD: ${{ env.ACR_PASSWORD }}
```

2. Replace `docker/login-action` + `docker/build-push-action` (or `oras`, `crane`) targets:

- Remove or stop using `ghcr.io`, `docker.io` login unless still required for mirrors.
- Add:

```yaml
- name: Login to Aliyun ACR
  if: needs.determine-build-context.outputs.push_enabled == 'true' # if your workflow uses this pattern
  uses: docker/login-action@b45d80f862d83dbcd57f89517bcf500b2ab88fb2 # pin by SHA; update intentionally
  with:
    registry: ${{ env.ACR_REGISTRY }}
    username: ${{ env.ACR_USERNAME }}
    password: ${{ env.ACR_PASSWORD }}
```

3. Update image references in `tags:` / `images:` to your ACR namespace and repository names.
4. Keep **provenance**, **SBOM**, and **digest pinning** behavior aligned with the existing workflow policy.

## 5) Deploy to Aliyun ECS (after ACR tags exist)

**Ordering:** ACR `latest` / `stable` (or channel) pointers must be applied in an earlier job (for example `post-release` → `release-push-to-channel`). ECS deploy runs **after** those jobs succeed so nodes pull the intended tag.

Example **caller** after a Docker release manifest (OpenClaw `docker-release.yml`): enable repo variable `ALIYUN_ECS_DEPLOY_ENABLED=true`, then:

```yaml
deploy-aliyun-ecs:
  name: Deploy to Aliyun ECS
  needs: [create-manifest]
  if: >-
    ${{
      always() &&
      needs.create-manifest.result == 'success' &&
      github.event_name != 'workflow_dispatch' &&
      vars.ALIYUN_ECS_DEPLOY_ENABLED == 'true'
    }}
  permissions:
    contents: read
    id-token: write
  uses: ./.github/workflows/deploy-aliyun-ecs.yml
  with:
    image_tag: latest
  secrets: inherit
```

Other repos may instead chain after `post-release` / `publish-to-aliyun-acr` if that is where `:latest` is set; keep **`needs:` order** so ECS runs only after the desired tag exists in ACR.

Implement `deploy-aliyun-ecs.yml` to:

- Pull Infisical secrets at `secret-path: /${{ github.event.repository.name }}` (same short-name rule as build jobs).
- Require deploy-time keys such as `ECS_HOST`, `ECS_USER`, `ECS_SSH_PRIVATE_KEY` (plus `ACR_REGISTRY` / `ACR_REPOSITORY` to build the pull ref).
- SSH to ECS (for example `appleboy/ssh-action`) and `docker pull` the resolved image; extend the remote script with `docker compose up -d` or host-specific steps as needed.
- Use the **same** image tag convention as publish (for example `latest` only when your release flow guarantees it).

## Checklist before merging workflow changes

- [ ] No plaintext secrets in YAML; OIDC identity and project slugs parameterized.
- [ ] `secret-path` matches the repo naming convention.
- [ ] Both `file` and `env` exports present only when each is required.
- [ ] ACR login uses env populated **after** Infisical fetch.
- [ ] ECS job `needs:` includes image publish + tag promotion jobs in correct order.
- [ ] Pinned action SHAs reviewed on upgrade (login-action example above is illustrative; align with repo pin policy).

## Related files (OpenClaw)

- `.github/workflows/docker-release.yml` — build/push ACR + manifest; optional ECS deploy job when `ALIYUN_ECS_DEPLOY_ENABLED=true`
- `.github/workflows/deploy-aliyun-ecs.yml` — reusable ECS deploy over SSH (`workflow_call`)

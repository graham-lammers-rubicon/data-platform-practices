# CI/CD and Deployment

Defines how code moves from branch to production. GitHub Actions is the CI/CD platform. Everything deployable is a bundle (DAB); deployment is automation-only above dev (see [Environments](environments.md)).

## What this covers

- Repo and branching standard
- The GitHub Actions pipeline and promotion gates
- Deployment identity (OIDC, no stored secrets)
- Rollback and hotfix

## Repos and branching

- One repo per domain bundle (`sales-pipelines`); infrastructure in its own Terraform repo.
- Trunk-based: short-lived feature branches, PR to `main`, no long-lived release branches.
- `prod` deploys only from `main`. Promotion is the same commit SHA moving `qa` → `test` → `prod`.

## Pipeline

| Trigger | Actions |
| --- | --- |
| Pull request | `databricks bundle validate` for every target, unit tests, lint |
| Merge to `main` | Deploy `qa`, run smoke job |
| Approval gate (GitHub environment) | Deploy `test`, run acceptance checks from the spec |
| Approval gate | Deploy `prod` |

Rules:

- CI uses the `databricks/setup-cli` action; deploys are `databricks bundle deploy -t <target>`.
- `qa`, `test`, `prod` targets use `mode: production` with `run_as` the deployment service principal and an explicit `permissions` block so engineers can view runs they do not own.
- `test` and `prod` are GitHub environments with required reviewers. Approval is recorded in GitHub, not in chat.
- One concurrency group per target: parallel deploys to the same target race and are blocked.
- Developers deploy to `dev` directly (`bundle deploy -t dev`, `mode: development`); dev is the only target a human deploys.
- Acceptance criteria come from the spec (see [Spec-driven development](../practices/spec-driven-development.md)); a deploy without a validation step is incomplete.

## Deployment identity

- Auth is OIDC workload identity federation: `DATABRICKS_AUTH_TYPE: github-oidc` with `DATABRICKS_HOST` and `DATABRICKS_CLIENT_ID`. No Databricks token is ever stored in GitHub secrets.
- One deployment SP per tier: `sp-dplat-deploy-np`, `sp-dplat-deploy-prod` (see [Naming conventions](naming-conventions.md), [Service principal authentication](service-principal-auth.md)).
- Each SP carries a federation policy scoped to this GitHub org, repo, and branch. The prod policy matches `refs/heads/main` only.

## Rollback and hotfix

- Bundles are declarative: rollback is redeploying the previous known-good SHA through the same pipeline. There is no manual rollback path.
- Hotfix: branch from `main`, PR with expedited review, same pipeline. Editing a deployed asset in the workspace is never the fix.

## Sharp edges

- A federation policy scoped to the org instead of repo-and-branch lets any workflow in the org mint tokens as the deploy SP. Scope every policy to the narrowest subject claim.
- Token-based auth (`SP_TOKEN` secret) works and is documented; here it is a defect. OIDC only.
- `bundle deploy` is not atomic across resources. A failed mid-deploy leaves mixed state; the fix is rerunning the deploy, never hand-editing the workspace.
- A killed production deploy can leave the deployment lock held; recover with `bundle deploy --force-lock`, never by manual cleanup.
- `bundle validate` catches config errors, not logic errors. A green validate proves the YAML parses and resolves, nothing more; tests and smoke runs carry the rest.
- Approval fatigue: if every merge pings the same two reviewers for `test`, gates get rubber-stamped. Keep the `test` gate cheap (auto-checks) and the `prod` gate human.

## Checklist

- [ ] PR runs validate for all targets plus tests; merge auto-deploys `qa`
- [ ] `test` and `prod` are GitHub environments with required reviewers
- [ ] Zero Databricks tokens in GitHub secrets; auth is `github-oidc`
- [ ] Federation policies scoped to repo and branch; prod to `main` only
- [ ] Concurrency group per target
- [ ] Every deployed asset traces to a commit SHA; rollback tested at least once
- [ ] Smoke and acceptance runs exist for every promotable bundle

## Sources

- Databricks: [GitHub Actions for CI/CD](https://docs.databricks.com/aws/en/dev-tools/ci-cd/github)
- Databricks: [CI/CD with Databricks Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/ci-cd-bundles)
- Databricks: [OAuth token federation](https://docs.databricks.com/aws/en/dev-tools/auth/oauth-federation)
- Databricks: [Deployment modes](https://docs.databricks.com/aws/en/dev-tools/bundles/deployment-modes)

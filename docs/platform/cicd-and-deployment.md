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
- `prod` deploys only from `main`. Promotion is the same commit SHA moving `nonprod` → `prod`.

## Pipeline

| Trigger | Actions |
| --- | --- |
| Pull request | `databricks bundle validate` for every target, unit tests, lint |
| Merge to `main` | Deploy `nonprod`, run smoke job and acceptance checks from the spec |
| Approval gate (GitHub environment `prod`) | Deploy `prod` |

Rules:

- CI uses the `databricks/setup-cli` action pinned to a version; deploys are `databricks bundle deploy -t <target>`.
- `nonprod` and `prod` targets use `mode: production` with `run_as` the deployment service principal and an explicit `permissions` block so engineers can view runs they do not own.
- `prod` is a GitHub environment with required reviewers and a deployment branch policy allowing `main` only. Approval is recorded in GitHub, not in chat.
- One concurrency group per target: parallel deploys to the same target race and are blocked.
- Developers deploy to `dev` directly (`bundle deploy -t dev`, `mode: development`); dev is the only target a human deploys.
- Acceptance criteria come from the spec (see [Spec-driven development](../practices/spec-driven-development.md)); a deploy without a validation step is incomplete.

## Deployment identity

- Auth is OIDC workload identity federation: `DATABRICKS_AUTH_TYPE: github-oidc` with `DATABRICKS_HOST` and `DATABRICKS_CLIENT_ID`. No Databricks token is ever stored in GitHub secrets.
- Deploy jobs set `permissions: id-token: write` (plus `contents: read`). Without it, GitHub refuses to issue the OIDC token and every deploy fails auth.
- One deployment SP per tier: `sp-dbx-deploy-np`, `sp-dbx-deploy-prod` (see [Naming conventions](naming-conventions.md), [Service principal authentication](service-principal-auth.md)).
- Each SP carries a federation policy (issuer `https://token.actions.githubusercontent.com`) scoped to the narrowest subject claim for this org and repo. The nonprod policy matches subject `repo:<org>/<repo>:ref:refs/heads/main`. The prod policy matches subject `repo:<org>/<repo>:environment:prod`; the `main`-only restriction comes from the environment's deployment branch policy, not the federation subject.

## Rollback and hotfix

- Bundles are declarative: rollback is redeploying the previous known-good SHA through the same pipeline. There is no manual rollback path.
- Hotfix: branch from `main`, PR with expedited review, same pipeline. Editing a deployed asset in the workspace is never the fix.

## Sharp edges

- A job that references a GitHub environment gets OIDC subject `repo:<org>/<repo>:environment:<name>`, not the branch form. A prod policy written against `refs/heads/main` never matches once the prod job uses the `prod` environment; the deploy fails auth. Scope the prod policy to the environment subject and enforce the branch in the environment settings.
- A federation policy scoped to the org instead of a full subject claim lets any workflow in the org mint tokens as the deploy SP. Scope every policy to the narrowest subject claim.
- Token-based auth (`SP_TOKEN` secret) works and is documented; here it is a defect. OIDC only.
- `bundle deploy` is not atomic across resources. A failed mid-deploy leaves mixed state; the fix is rerunning the deploy, never hand-editing the workspace.
- A killed production deploy can leave the deployment lock held; recover with `bundle deploy --force-lock`, [documented](https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands) for use "only if the previous deployment crashed or was interrupted and left a stale lock file," never by manual cleanup.
- `bundle validate` catches config errors, not logic errors. A green validate proves the YAML parses and resolves against the schema, nothing more; tests and smoke runs carry the rest.
- Approval fatigue: there is one human gate (`prod`). Keep it that way. The `nonprod` stage proves the SHA with automated checks; adding reviewers to it buys rubber stamps, not safety.

## Checklist

- [ ] PR runs validate for all targets plus tests; merge auto-deploys `nonprod` with smoke and acceptance checks
- [ ] `prod` is a GitHub environment with required reviewers and a `main`-only branch policy
- [ ] Zero Databricks tokens in GitHub secrets; auth is `github-oidc` with `id-token: write` on deploy jobs
- [ ] Federation policies scoped per tier: nonprod to repo and `main` branch, prod to the `prod` environment subject
- [ ] Concurrency group per target
- [ ] Every deployed asset traces to a commit SHA; rollback tested at least once
- [ ] Smoke and acceptance runs exist for every promotable bundle

## Sources

- Databricks: [GitHub Actions for CI/CD](https://docs.databricks.com/aws/en/dev-tools/ci-cd/github)
- Databricks: [Enable workload identity federation for GitHub Actions](https://docs.databricks.com/aws/en/dev-tools/auth/provider-github)
- Databricks: [Deployment modes](https://docs.databricks.com/aws/en/dev-tools/bundles/deployment-modes)
- Databricks: [Bundle command group](https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands) (`--force-lock`, `validate` scope)
- GitHub: [Security hardening with OpenID Connect](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- GitHub: [Managing environments for deployment](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)
- GitHub: [Concurrency](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs)

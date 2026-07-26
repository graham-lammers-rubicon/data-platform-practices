# Onboarding: Platform Engineer

You own the Azure and Databricks infrastructure the platform runs on: provisioning, networking, identity, compute governance, and deployment automation. This page is your reading order and first-week checklist.

## Read in this order

1. [Access model](../governance/access-model.md) - the grants you will be implementing in Unity Catalog
2. [Azure infrastructure](../platform/azure-infrastructure.md) - subscriptions, resource groups, workspaces, storage, networking, identity
3. [Environments](../platform/environments.md) - environment topology and promotion
4. [CI/CD and deployment](../platform/cicd-and-deployment.md) - bundles, pipelines, deployment identity
5. [Compute policies](../platform/compute-policies.md), [Naming conventions](../platform/naming-conventions.md), [Secrets and credentials](../platform/secrets-and-credentials.md), [Service principal authentication](../platform/service-principal-auth.md)
6. [Medallion data practices](../practices/medallion-data-practices.md) - skim the access matrix and pipeline architecture sections; the infrastructure exists to enforce them

## First-week checklist

- [ ] Azure subscription and Databricks account access granted
- [ ] Map the current resource group and workspace layout against the [Azure infrastructure](../platform/azure-infrastructure.md) doc; file gaps as issues
- [ ] Verify Unity Catalog grants match the [access matrix](../governance/access-model.md); flag any consumer with Bronze or Silver access as a defect
- [ ] Confirm every workspace has cluster policies enforcing the [compute policies](../platform/compute-policies.md)
- [ ] Run one bundle deployment through the promotion path end to end

## Rules you will be held to

- Infrastructure is provisioned as code. Portal-created resources are a defect.
- Least privilege by default; grants on groups, never individual users.
- No secrets in code or docs. Key Vault-backed scopes and service principals only.
- Cost is governed: right-sized SKUs, auto-termination, tagging for attribution.
- The platform docs are normative. When you establish a new standard, write it into the owning doc in the same change; the doc is the deliverable, not a wiki page elsewhere.

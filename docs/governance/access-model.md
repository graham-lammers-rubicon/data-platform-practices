# Access Model

Defines who can read and write each medallion layer, how access is requested, and how it is reviewed. This is the authoritative home of the access matrix.

## What this covers

- The role-by-layer access matrix
- The rules behind it
- How identities and groups are provisioned
- How to request access and how requests are reviewed

## Access matrix

The matrix applies per environment (`dev_catalog`, `nonprod_catalog`, `prod_catalog`). Where a cell differs by environment, the environment is stated.

| Role | Bronze | Silver | Gold |
| --- | --- | --- | --- |
| Domain pipeline SP (`sp-<domain>-pipeline-<env>`) | READ/WRITE (own domain) | READ/WRITE (own domain); READ conformed dimensions | READ/WRITE (own domain) |
| Conformed-dimensions pipeline SP | READ (its sources) | READ/WRITE (dimension tables) | none |
| Cross-domain Gold job SP (`sp-<subject>-gold-<env>`) | none | READ (named source domains) | READ/WRITE (its objects) |
| Data engineers | dev: READ/WRITE. nonprod/prod: READ | dev: READ/WRITE. nonprod/prod: READ | dev: READ/WRITE. nonprod/prod: READ |
| Analysts / data scientists | none | READ (approved, prod) | READ |
| Consuming services | none | none | READ (prod) |

- Human WRITE exists only in `dev_catalog`, across all three layers, through a developer's own `mode: development` deploys. Dev objects are disposable: user-owned, user-prefixed, never promoted in place, and subject to cleanup. In `nonprod` and `prod`, every write path is a service principal, so every permanent entity is SP-owned; this is the same rule as "Human identities do not hold production WRITE" below.
- A pipeline SP's WRITE means `USE SCHEMA` plus `CREATE TABLE` on the layer schemas; it owns the tables its pipeline creates and holds no MODIFY on other domains' tables. Ownership is the write-isolation boundary between domains sharing a layer schema.
- A cross-domain Gold job declares the Silver domains it reads in its spec; its SP is granted exactly those.

## Rules

- Access is least privilege by default: requested per role and per layer, never granted broadly.
- Downstream consumers never get Bronze or Silver access. Gold is the only layer consuming services touch: analytics, GenAI retrieval, APIs, MCP servers. Any consumer connection to Bronze or Silver is a defect, not an exception.
- Analyst Silver access requires approval and is READ only. It exists for profiling and validation work, not for building consumer-facing outputs.
- Production writes go through pipeline service principals only. Human identities do not hold production WRITE.
- All grants live in Unity Catalog on groups, not individual users. Audit comes from UC system tables, not spreadsheets.
- Every grant is declared in the Terraform repo and applied by CI. A grant issued through the UI or an ad hoc `GRANT` statement is a defect, even when the access itself is correct; re-issue it through code.
- Self-service means exploring certified Gold data. Direct business-user access to raw tables is not self-service; it is distributed data engineering without standards (see [BI practices guidance](../practices/bi-practices-guidance.md)).
- Grants are coarse; sensitive data needs fine-grained controls on top. Columns classified as PII carry Unity Catalog column masks for non-privileged groups, and row filters scope rows where a role should see only a subset. This matters most for prod data readable from nonprod through the read-only binding: the binding blocks writes, masks are what limit what a nonprod reader sees. Classification tiers and which ones require masks: [Data lifecycle](data-lifecycle.md) (decision open).
- Masks and filters are code: defined in the repos and applied by CI like every other grant. A mask added in the UI is the same defect as a UI grant.

## Identity provisioning

- Automatic identity management (AIM) is the provisioning mechanism. Microsoft Entra ID is the source of record for users and groups; [AIM is enabled by default](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/automatic-identity-management/) for accounts created after 2025-08-01, and Databricks recommends it over SCIM.
- The Entra SCIM provisioning connector is not used. Databricks [warns that mixing provisioning methods](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/automatic-identity-management/) "causes duplicate entries and permission conflicts"; never stand SCIM up.
- Grants attach to Entra-synced groups (`grp-<role>-<scope>-<env>`, see [Naming conventions](../platform/naming-conventions.md)). Membership is managed in Entra, never edited in Databricks.
- Service principals are not provisioned by AIM sync alone: Databricks-managed SPs are created by Terraform ([Service principal authentication](../platform/databricks-service-principal-auth.md)); Entra SPs provision on first authentication.
- Nested group members inherit permissions, but principals not explicitly provisioned to the account are invisible to Terraform and the APIs. Anything referenced in code is explicitly provisioned.

## Requesting access

Two request shapes exist. Both leave an auditable trail; neither is a chat message.

**Joining an existing role** (the common case): request membership in the matching `grp-<role>-<scope>-<env>` Entra group. Approval is the only human step: the group owner approves, provisioning is automatic from there (AIM syncs membership on the next authentication; see [Identity provisioning](#identity-provisioning)). Owners: domain groups by the data owner, platform groups by the platform team. Target: same business day.

**A new access shape** (new group, new grant, or an exception like analyst Silver access): a PR to the infrastructure repo adding the group or grant in Terraform, linking the justification (the spec or decision it serves). CI posts the plan on the PR; the data owner for the scope and a platform engineer approve; merge applies automatically. Exceptions are time-boxed in the PR with an expiry date. Target: next business day. If turnaround exceeds that, the fix is automation or approver coverage, not a longer target.

Review and revocation:

- Quarterly access review: group memberships against role rosters, and the Terraform grant set against the access matrix. Findings are PRs, not notes.
- Offboarding is automatic for identity: Entra removal deactivates the user via AIM. Token revocation is the explicit extra step ([sharp edges](#sharp-edges)).
- A grant whose purpose ended is removed by PR when the owning product retires, not left until the next review finds it.
- Break-glass access (incident response) is granted by the platform team, time-boxed, and reviewed at the next working day; the audit trail is `system.access.audit`.

## Sharp edges

- Granting an analyst Silver access "temporarily" for one report creates a permanent dependency. If a consumer needs a Silver measure, the fix is a Gold object, not a grant.
- Grants to individual users survive team changes silently. Group-based grants are the only auditable pattern.
- Renaming a group in Entra does not sync proactively; the name updates only when an admin opens the group detail page. Treat group names as immutable once a grant exists.
- Deleting a user in Entra deactivates them in Databricks but does not revoke their personal access tokens; [Databricks recommends revoking tokens](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/automatic-identity-management/) for deactivated identities. Token revocation is an explicit offboarding step.
- AIM does not support cross-tenant Entra directories. External collaborators need a different path; do not assume B2B guests sync.

## Checklist

- [ ] Every grant maps to a row in the access matrix
- [ ] Every grant exists in Terraform state; UC system tables show no grants issued outside CI identities
- [ ] No consuming service has any Bronze or Silver grant
- [ ] All grants are on groups, not users
- [ ] Production WRITE is held only by pipeline service principals; dev objects are user-owned and disposable
- [ ] PII-classified columns carry column masks; masks and row filters exist in code, not UI
- [ ] AIM is enabled; no SCIM connector exists in the Entra tenant for Databricks
- [ ] Every granted group is Entra-synced with membership managed in Entra

## Sources

- Azure Databricks: [Automatic identity management](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/automatic-identity-management/)
- Azure Databricks: [Sync users and groups from Microsoft Entra ID using SCIM](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/scim/)
- Azure Databricks: [Service principals](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/service-principals)

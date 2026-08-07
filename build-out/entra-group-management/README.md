# Entra Group Management

Build-out working material for creating and managing the platform's Microsoft Entra ID groups: the group inventory, membership sources, and the configuration-as-code that provisions them.

Governing standards (docs/ wins on conflict):

- [Access model](../../docs/governance/access-model.md) - which roles get which access per layer and environment; grants attach to groups, never users.
- [Naming conventions, Identity section](../../docs/platform/naming-conventions.md) - group pattern `grp-<role>-<scope>-<env>` (e.g. `grp-analysts-gold-prod`); no vendor or implementer tokens.
- [Azure infrastructure, Identity section](../../docs/platform/azure-infrastructure.md) - identity wiring and the Terraform standard; groups are provisioned as code, a portal-created group is a defect.

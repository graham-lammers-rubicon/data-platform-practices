# Resilience and Disaster Recovery

> **Status: stub.** Scope and required decisions are agreed; the answers are not. This doc is not normative until the decisions below are made and the notice is removed. Decisions are made with the Cloud and DevOps teams.

Defines the platform's resilience posture: what survives what failure, at what cost, decided on purpose rather than by default.

## Current posture (explicit, not accidental)

- Single region, single metastore, by design. Region expansion is an architecture change ([Azure infrastructure](azure-infrastructure.md)).
- No committed RTO/RPO exists yet. Until one does, no output should be certified with an availability promise.

## Decisions to make

Each requires an owner, a decision, and a date. Answers open.

| Decision | Question | Owner |
| --- | --- | --- |
| RTO/RPO per tier | What outage duration and data loss is acceptable for prod? For nonprod? | TBD |
| Storage redundancy | LRS, ZRS, or GRS for the tier storage accounts? Cost vs survivability. | TBD |
| Delta data recovery | Backup mechanism and cadence for UC managed tables (deep clone, replication)? What is the tested restore path? | TBD |
| Metastore and workspace loss | Recovery procedure if the metastore or a workspace is lost. Terraform re-create covers config; what covers data and grants state? | TBD |
| Terraform state | Backup and recovery for the state storage account. | TBD |
| Key Vault | Soft-delete and purge protection required? Recovery procedure for a deleted vault backing a scope. | TBD |
| Region loss | Accepted risk with a revisit trigger, or a warm-standby design? If accepted: who signed, and what event reopens the decision? | TBD |
| Test cadence | How often is restore actually exercised? An untested backup is a hope. | TBD |

## Checklist (activates when decisions land)

- [ ] RTO/RPO documented per tier and reflected in certified-output claims
- [ ] Storage redundancy tier set in Terraform, not defaulted
- [ ] Restore procedure documented and tested at the declared cadence
- [ ] Region-loss decision signed by a named owner with a revisit trigger

## Sources

- To be added when decisions are made. Candidates: Azure Databricks disaster recovery guidance, Azure storage redundancy documentation.

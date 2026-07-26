# Responsible Use

Defines what acceptable use of the platform's data and compute means, and how use is monitored. The platform grants capability; this doc states the obligations that come with it. Access rules live in the [Access model](access-model.md); this doc governs behavior after access is granted.

## What this covers

- Data handling obligations
- Compute use and cost accountability
- GenAI and agent use of platform data
- Monitoring, audit, and escalation

## Data handling

- Data is used for the purpose its access was granted. A grant to build one product is not a license to browse the domain. Re-sharing data outside the granted role, by export, screenshot, or copy, is a violation, not a workaround.
- Production data read from nonprod through the read-only binding is still production data. Handling obligations follow the data, not the workspace ([Environments](../platform/databricks-environments.md)).
- Exports to spreadsheets or personal storage to "get the numbers" are a defect signal, instrumented and reviewed ([BI practices guidance](../practices/bi-practices-guidance.md)). The sanctioned path is certified Gold through governed tools.
- Bronze and Silver data stays inside the platform: no examples, tickets, screenshots, or external tools. Gold is shared only within its granted audience.
- Real data never appears in docs, code comments, commit messages, or test fixtures. Test data is provisioned snapshots inside the platform ([Environments](../platform/databricks-environments.md)), never copies carried out of it.

## Compute use

- Platform compute runs platform workloads. Personal experimentation happens in dev under the interactive policy with auto-termination; anything long-running or scheduled needs an owner and a tag trail ([Compute policies](../platform/databricks-compute-policies.md)).
- Spend is accountable: every DBU traces to a domain and owner through tags. Working around the policy set to get untagged compute is a violation even when the work itself is legitimate.
- Unpausing a nonprod schedule requires a stated reason ([Environments](../platform/databricks-environments.md)).
- Cost discretion for individual runs (estimating before launch, killing overruns, right-sizing): [Compute policies](../platform/databricks-compute-policies.md#cost-discretion-by-individuals).

## GenAI and agents

- GenAI retrieval, agents, and MCP servers read Gold only, like every consuming service. An agent with Bronze or Silver access is the same defect as any other consumer connection to those layers.
- Answers trace to sources: retrieval carries provenance (source object, retrieved context ids). An assistant that cannot cite its Gold sources is not decision-grade.
- Secrets, credentials, and confidential data do not go into prompts to tools outside the governed platform. The same export rule applies to context windows as to spreadsheets.
- Agents operating on this platform follow the repo's standards and cite the doc being applied (see the repo's audience statement); an agent that invents patterns bypasses governance.

## Monitoring and audit

- Unity Catalog system tables are the audit source: [`system.access.audit`](https://learn.microsoft.com/en-us/azure/databricks/admin/system-tables/audit-logs) for access and identity events, `system.billing.usage` for spend. Audit questions are answered with queries, not recollection.
- Access reviews run per the [Access model](access-model.md) cadence; usage anomalies (broad scans outside a granted purpose, export spikes, untagged spend) are reviewed by the platform team.
- Monitoring exists to keep the platform trustworthy, not to surveil individuals. Findings route to the owning team first.

## Escalation

- Suspected data exposure, a pushed secret, or a wrong grant: report to the platform team immediately. For secrets, rotation precedes cleanup ([Secrets and credentials](../platform/secrets-and-credentials.md)); for grants, revocation precedes root-cause.
- A certified output found wrong: freeze its certification, notify the named decision-maker from its spec, then fix. Decisions made on bad certified numbers are the platform's highest-severity incident class.
- Honest self-reporting of a mistake is the expected path. The violation is concealment, not error.

## Sharp edges

- The read-only prod binding makes prod PII one grant away from every nonprod user. Purpose-bound grants and this doc's handling rules are the control; the binding alone is not ([Environments](../platform/databricks-environments.md)).
- Audit tables prove what happened; they do not prevent it. Prevention lives in grants, bindings, and policies. Treating audit as the control means discovering violations months late.
- An agent given a human's broad credentials inherits the human's entire blast radius. Agents get scoped service principals, never borrowed sessions.

## Checklist

- [ ] Every standing data use maps to a granted purpose
- [ ] Zero consumer or agent connections to Bronze or Silver
- [ ] Export monitoring active; findings reviewed with the owning team
- [ ] No real data in docs, tickets, fixtures, or external tools
- [ ] Audit queries for access and spend are runnable by the platform team on demand
- [ ] Escalation contacts published; last incident followed the documented path

## Sources

- Azure Databricks: [Audit log system table reference](https://learn.microsoft.com/en-us/azure/databricks/admin/system-tables/audit-logs)
- Internal: [Access model](access-model.md), [Environments](../platform/databricks-environments.md), [BI practices guidance](../practices/bi-practices-guidance.md), [Secrets and credentials](../platform/secrets-and-credentials.md)

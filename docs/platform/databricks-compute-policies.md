# Compute Policies

Defines compute governance: which compute type each workload class uses, the policies that enforce sizing and termination, and how every DBU traces to an owner. Compute policies require the Premium plan; this platform is on it.

## What this covers

- Individual cost discretion: the spend judgment policies cannot enforce
- Workload classes and their compute types
- The policy set and what each policy fixes
- Cost attribution: tags, serverless usage policies, budgets
- Compliance enforcement when policies change

## Cost discretion by individuals

Conduct obligations (purpose-bound compute, tagging workarounds, self-reporting) live in [Responsible use](../governance/responsible-use.md). This section is the judgment policies cannot enforce: whether a run is a good use of money. Serverless bills per second; what you run is what the domain pays for, and your usage policy attributes it to you.

- Know your cost before you launch. A full refresh, a large backfill, or a high-frequency schedule is a spend decision; size the run to the need and prefer incremental over full recomputes.
- Long-running jobs and notebooks burn DBUs for their entire duration. Always use discretion: estimate the runtime before launching, check on runs that should have finished, and kill anything far past its estimate instead of letting it complete.
- You stop what you start. An idle warehouse or a forgotten continuous run is your spend until it is stopped.
- Right-size warehouses and let auto-stop work. Do not resize a shared warehouse for a one-off query; use a smaller one or ask the platform team.
- Schedules are standing costs. A schedule nobody reads output from is pure waste and gets removed, not tolerated.
- Your spend is visible: `system.billing.usage` filtered by your tags shows what you cost, per day. Check it when you change how you work; attributed but unjustified spend is yours to explain.

## Workload classes

Serverless is the preferred default, not a hard requirement. The standard workspaces are hybrid: VNet-injected with serverless enabled via NCC ([Azure infrastructure](azure-infrastructure.md)), so classic compute is available in every workspace, always behind a Terraform-defined policy. A workload choosing classic compute (for example CDC/JDBC pull ingestion from source databases over the private network path) records the justification in its spec.

| Workload | Compute | Governance |
| --- | --- | --- |
| Lakeflow SDP pipelines | Serverless | Serverless usage policy tags; continuous mode is prod-only ([Environments](databricks-environments.md)) |
| Jobs and pull ingestion | Serverless; classic via `jobs-standard` policy where the workload needs the classic network path | Policy or usage policy |
| Interactive notebooks | Serverless | Usage policy |
| SQL warehouses | Serverless, auto-stop | Sized per team: `<team>-<size>` ([Naming conventions](naming-conventions.md)) |
| Vector search, model serving | Serverless endpoints | Serverless usage policy tags |

## Rules

- Every classic compute resource is created through a policy. No engineer holds the unrestricted-cluster-creation entitlement; the built-in Unrestricted policy is reachable only by workspace admins, and using it is a defect outside break-glass work.
- Policies are Terraform-defined per the everything-as-code rule ([Environments](databricks-environments.md)), named by workload class (`jobs-standard`, `interactive-standard`). A policy edited in the UI is configuration drift.
- Policies start from [Databricks policy families](https://learn.microsoft.com/en-us/azure/databricks/admin/clusters/policy-families) and fix, at minimum: auto-termination (interactive compute), a max DBUs-per-hour ceiling, an allowed node-type list, max compute resources per user, and the required tag set.
- The baseline tags (`env`, `domain`, `owner`, `costCenter`, `managedBy`) are enforced in the policy's tag rules. An untagged cluster should be impossible to create.
- Serverless usage is attributed with [serverless usage policies](https://learn.microsoft.com/en-us/azure/databricks/admin/usage/budget-policies) (Public Preview): one per domain, carrying the same baseline tags, which propagate to `system.billing.usage.custom_tags` and Azure cost analysis. Assign each engineer exactly one policy so it applies automatically.
- Fallback while usage policies are Preview: attribution by identity. `system.billing.usage` joined on the run-as identity maps every DBU to a principal, and principals map to domains by naming convention. If the preview changes or its gaps persist, this query-based attribution is the backstop, not a return to untracked spend.
- Cost review is standing, not reactive: budgets with alerts in the account console, and `system.billing.usage` queries by tag. Spend that cannot be attributed to a domain is a defect in the policy set, not a finance problem.
- After any policy change, run compliance enforcement on the governed compute. Policy edits do not propagate to existing resources on their own.
- Library standardization uses policy-attached libraries, not init scripts; Databricks [recommends compute policies over init scripts](https://learn.microsoft.com/en-us/azure/databricks/admin/clusters/policies) for library installs.
- Nonprod compute follows the trigger rules in [Environments](databricks-environments.md): manual or CI trigger, no standing schedules without a stated reason.

## Sharp edges

- Editing a policy changes nothing that already exists: "compute resources created using that policy aren't automatically updated." Enforce compliance after every policy change or the fleet drifts from the policy silently.
- Deleting a policy leaves its compute running but uneditable except by unrestricted-creation holders, which nobody has. Migrate compute to a replacement policy before deleting.
- Lowering max-resources-per-user does not terminate existing resources over the limit; they run until manually stopped.
- Serverless usage policies do not cover classic compute, do not auto-attach to existing assets, and a job-triggered pipeline does not inherit the job's policy. Each of those gaps is untagged spend until closed by hand.
- A user assigned multiple serverless usage policies gets the alphabetically first one by default; the tag data then lies about ownership. One policy per engineer.
- Policy-attached libraries remove user-installed compute-scoped libraries on the next restart. Intended, but it surprises anyone who installed from the UI.

## Checklist

- [ ] Every classic compute resource was created through a Terraform-defined policy
- [ ] No non-admin identity has unrestricted cluster creation
- [ ] Policies fix auto-termination, DBU ceiling, node types, max resources per user, and required tags
- [ ] Every engineer has exactly one serverless usage policy; every domain's serverless spend is tagged
- [ ] Engineers can run the `system.billing.usage` query for their own tags; nonprod shows no unowned schedules or idle standing compute
- [ ] `system.billing.usage` shows zero unattributable spend
- [ ] Budgets with alerts exist per domain and per tier
- [ ] Compliance enforcement ran after the last policy change

## Sources

- Azure Databricks: [Create and manage compute policies](https://learn.microsoft.com/en-us/azure/databricks/admin/clusters/policies)
- Azure Databricks: [Default policies and policy families](https://learn.microsoft.com/en-us/azure/databricks/admin/clusters/policy-families)
- Azure Databricks: [Serverless usage policies](https://learn.microsoft.com/en-us/azure/databricks/admin/usage/budget-policies)
- Azure Databricks: [Create and monitor budgets](https://learn.microsoft.com/en-us/azure/databricks/admin/account-settings/budgets)

# Data Platform Practices

Documentation for a Databricks-based Data Intelligence platform hosted in Azure. This repo is normative: it defines the standards the team builds and operates against.

It is written for humans and agents. Agents such as Claude Code and Databricks Genie SHOULD use it as the baseline guidance when building on this platform: follow the rules in these docs before inventing patterns, and cite the doc being applied.

## Start here by role

- [Data Engineer](onboarding/data-engineer.md)
- [BI Analyst](onboarding/bi-analyst.md)
- [Platform Engineer](onboarding/platform-engineer.md)
- Not building? Decision-makers, architects, and reviewers: read [BI practices, section 1](practices/bi-practices-guidance.md) for what the platform is for, the [access model](governance/access-model.md) for who may touch what, and the [practices arc](practices/index.md) for how the standards fit together.
- Agents: [CLAUDE.md](../CLAUDE.md) is the distilled ruleset; these reference docs win when they disagree.

## Sections

Each section answers a different question:

- [Onboarding](onboarding/index.md) - *where do I start?* Role-based entry paths with reading orders and first-week checklists.
- [Practices](practices/index.md) - *what are the rules for what I build?* The design and delivery standards, arranged as an arc: shape the data, deliver the work, serve decisions.
- [Platform](platform/index.md) - *how does the platform itself run?* Azure, Databricks, and GitHub reference, grouped by intent: stand it up, secure it, serve the semantic layer.
- [Governance](governance/index.md) - *what am I allowed to do?* Access, usage obligations, and the open data-lifecycle decisions.

Architecture decisions and their rationale are indexed in the [platform decision register](platform/decision-register.md); open items live there with owners.

Docs marked **Status: stub** have agreed scope but unwritten rules; they are not normative until the notice is removed.

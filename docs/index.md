# Data Platform Practices

Documentation for a Databricks-based Data Intelligence platform hosted in Azure. This repo is normative: it defines the standards the team builds and operates against.

It is written for humans and agents. Agents such as Claude Code and Databricks Genie SHOULD use it as the baseline guidance when building on this platform: follow the rules in these docs before inventing patterns, and cite the doc being applied.

## Start here by role

- [Data Engineer](onboarding/data-engineer.md)
- [BI Analyst](onboarding/bi-analyst.md)
- [Platform Engineer](onboarding/platform-engineer.md)

## Sections

- [Onboarding](onboarding/index.md) - role-based entry paths for new team members
- [Practices](practices/index.md) - design and delivery standards: medallion architecture, dataset contracts, BI standards, tidy data, spec-driven development
- [Platform](platform/index.md) - infrastructure reference: Azure footprint, environments, compute, naming, secrets, CI/CD
- [Governance](governance/index.md) - access model and responsible use

Docs marked **Status: stub** have agreed scope but unwritten rules; they are not normative until the notice is removed.

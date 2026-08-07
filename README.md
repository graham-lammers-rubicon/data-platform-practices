# data-platform-practices

Normative standards for a Databricks Data Intelligence Platform on Azure: how data is shaped, delivered, governed, and served. This repo is the rulebook; worked examples live in the reference domain repos.

Written for humans and agents.

- **Humans**: start at [docs/index.md](docs/index.md) and follow your role path (data engineer, BI analyst, platform engineer). Not building? Read [BI practices, section 1](docs/practices/bi-practices-guidance.md) for what the platform is for, and the [access model](docs/governance/access-model.md) for who may touch what.
- **Agents** (Claude Code, Databricks Genie): [CLAUDE.md](CLAUDE.md) is the distilled ruleset. Follow these docs before inventing patterns and cite the doc being applied. When CLAUDE.md and a reference doc disagree, the reference doc wins.

Docs marked **Status: stub** have agreed scope but open decisions; they are not normative until the notice is removed.

Changes go through PR. A rule change updates the reference doc first, then the CLAUDE.md distillation.

## Vendor-driven work items: Nimble Gravity / WS04

Workstream 04 (Data Foundation Build) runs with Nimble Gravity (NG lead: Dave Newman; client counterpart: Graham Lammers). The 2026-08 architecture conflicts between NG's plans and this repo are decided and recorded in the [platform decision register](docs/platform/decision-register.md). The items below originate in NG's WS04 documents and are NG-driven; they are tracked here for visibility, but any resolution that touches platform standards lands as a PR to this repo, not as a vendor document.

- **Source system inventory** - full inventory including GUS and other operational/reporting systems; confirmed so far: Azure SQL DB, Amazon RDS, Amazon S3 pending the Palantir Foundry investigation.
- **Source onboarding sequence** - priority order for onboarding sources within the Build phase.
- **AWS environment handoff** - documented handoff of AWS/Terraform knowledge from the exiting vendor (Rackspace); gating dependency for migration steps.
- **AWS-to-Azure connectivity model** - VPN/ExpressRoute vs public endpoint with allowlist vs cross-cloud PrivateLink vs landing data behind an Azure endpoint, for RDS/S3 ([register open item](docs/platform/decision-register.md#open-items)).
- **Azure SQL DB network location** - where the existing source lives and its reconciliation into the hub-and-spoke topology ([register open item](docs/platform/decision-register.md#open-items)).
- **IP address plan validation** - hub and spoke CIDRs against AWS VPC ranges and existing Rubicon networks ([register open item](docs/platform/decision-register.md#open-items)).
- **Azure region confirmation** - pending tenant review ([register open item](docs/platform/decision-register.md#open-items)).
- **OneLake mirroring evaluation** - for the Power BI connection to Gold.
- **Data quality and reconciliation thresholds** - tolerance thresholds co-defined with Rubicon business stakeholders during the Build-to-Certify transition.

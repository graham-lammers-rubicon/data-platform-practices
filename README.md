# data-platform-practices

Rubicon is building a Databricks Data Intelligence Platform: a governed lakehouse hosted on Azure where data is ingested once, shaped through medallion layers, governed in Unity Catalog, and served from a certified semantic layer to BI, analytics, and AI-driven consumption. The platform is being stood up under Workstream 04 (Data Foundation Build) with our partner Nimble Gravity. This repo is the working home for that effort: the standards the platform is built and operated against, and the working material for the build itself.

The ask of every reader, Rubicon or partner, human or agent, is the same: read the standards before you build, follow them instead of inventing patterns, and when a standard is wrong or missing, change it by branch and PR rather than working around it. Nothing here is frozen; everything here is authoritative until changed.

The repo serves two purposes:

1. **Platform guidelines** ([docs/](docs/index.md)) - the lasting, normative standards for how data is shaped, delivered, governed, and served: practices, platform reference, and governance policy. This is the rulebook the team builds and operates against; worked examples live in the reference domain repos.
2. **Build-out** ([build-out/](build-out/README.md)) - working material for Workstream 04 (Data Foundation Build), which stands up the Databricks platform: workspaces, Unity Catalog wiring, compute, CI/CD via Databricks Asset Bundles, and the ingestion and pipeline foundations, plus the Azure footprint Databricks requires underneath (subscriptions, network, storage, identities). Holds Nimble Gravity's WS04 plans, the vendor-driven work items, and the mapping from vendor proposals to the governing docs. Not normative; where it conflicts with docs/, docs/ wins, and the outcomes are recorded in the [platform decision register](docs/platform/decision-register.md).

## Goals

- One authoritative place for platform standards, so definitions and patterns do not diverge across teams, vendors, and tools.
- Decisions recorded with rationale in the [decision register](docs/platform/decision-register.md), so the "why" survives the engagement that produced it.
- Guidance executable by both humans and agents without additional context.

## Who reads this

- **Humans**: start at [docs/index.md](docs/index.md) and follow your role path (data engineer, BI analyst, platform engineer). Not building? Read [BI practices, section 1](docs/practices/bi-practices-guidance.md) for what the platform is for, and the [access model](docs/governance/access-model.md) for who may touch what.
- **Agents** (Claude Code, Databricks Genie): [CLAUDE.md](CLAUDE.md) is the distilled ruleset. Follow these docs before inventing patterns and cite the doc being applied. When CLAUDE.md and a reference doc disagree, the reference doc wins.

## This is a living repo

These documents are working documents, not a frozen publication. Corrections, modifications, and requests are welcome from anyone on the team or from partners, and all follow the same branch-and-PR process:

1. Branch from `main`.
2. Make the change. A rule change updates the reference doc first, then the CLAUDE.md distillation; a decision gets its register row and its normative content in the same PR.
3. Open a PR to `main` for review. The PR is the change record; a doc edit that bypasses review is a defect, the same as a UI-made infrastructure change.

Requests without a proposed edit are fine too: open an issue or a stub PR describing the gap, and the owning team picks it up.

Docs marked **Status: stub** have agreed scope but open decisions; they are not normative until the notice is removed.

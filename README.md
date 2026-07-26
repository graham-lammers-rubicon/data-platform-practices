# data-platform-practices

Normative standards for a Databricks Data Intelligence Platform on Azure: how data is shaped, delivered, governed, and served. This repo is the rulebook; worked examples live in the reference domain repos.

Written for humans and agents.

- **Humans**: start at [docs/index.md](docs/index.md) and follow your role path (data engineer, BI analyst, platform engineer). Not building? Read [BI practices, section 1](docs/practices/bi-practices-guidance.md) for what the platform is for, and the [access model](docs/governance/access-model.md) for who may touch what.
- **Agents** (Claude Code, Databricks Genie): [CLAUDE.md](CLAUDE.md) is the distilled ruleset. Follow these docs before inventing patterns and cite the doc being applied. When CLAUDE.md and a reference doc disagree, the reference doc wins.

Docs marked **Status: stub** have agreed scope but open decisions; they are not normative until the notice is removed.

Changes go through PR. A rule change updates the reference doc first, then the CLAUDE.md distillation.

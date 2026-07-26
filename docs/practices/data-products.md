# Data Products

**Last updated:** March 2026

---

## Definition

A **data product** on this platform is a well-defined, independently deliverable unit of data and logic that encodes our domain expertise, packaged for reliable, repeated use across applications, services, and intelligent agents.

A platform positioned as the **system of record for its domain** demands more than raw data storage. A data product is how domain expertise is operationalized: encapsulated once, governed, and consumable by any downstream system (a customer API, an AI agent, an MCP tool, or another data product) without being recreated each time.

---

## Core Principles

Every data product on this platform must be:

### Customer Centric

A consumer — whether a developer, an AI agent, or an API client — should not need knowledge of internal pipelines to use it. The interface is clean, the schema is documented, and the semantics are unambiguous.

### Composable

Each dataset is a self-contained unit that connects cleanly with others. A risk signal built on an entity master can be joined with a related structural dataset without coupling their internals.

### Trusted

The data product is the authoritative source for its domain. Quality is measured, lineage is documented, and freshness guarantees are explicit. Consumers rely on it to power products, make decisions, and inform agents — with no guesswork.

### Interoperable

A data product is designed to work across systems, tools, and consumers without custom integration for each. Its schema, semantics, and interfaces follow shared standards so that any downstream system — whether an API, an AI agent, an MCP tool, or another data product — can consume it without translation layers or tribal knowledge. Field definitions, units, classifications, and business rules are embedded in the product itself, not assumed to exist elsewhere.

### Extendable

New attributes, signals, or risk factors can be added without breaking existing consumers. Schemas evolve in a controlled way, and domain experts layer in intelligence over time without rebuilding from scratch.

---

## What a Data Product Is Not

| This is a data product | This is not |
| --- | --- |
| Curated entity master with documented schema and SLAs | A raw ingestion table with no ownership |
| A risk profile with versioned lineage | A one-off analyst query |
| A lifecycle signal feed with explicit freshness guarantees | An ad-hoc dashboard export |
| An authoritative risk score with documented logic | A derived column added to a table without documentation |
| A Gold view bounded by its subject area, composable with others | A view shaped or optimized for a specific consumer or use case |

---

## Anatomy of a Data Product

Each data product is defined by a standard set of attributes:

| Attribute | Description |
| --- | --- |
| Domain | The business subject area (e.g., entity master, product structure, lifecycle status, risk signals, substitutions) |
| Owner | The team or individual accountable for data quality, freshness, and evolution |
| Schema | A documented, versioned data model with field-level descriptions and data types |
| Lineage | Where the data originates and how it is transformed — from raw source to published product |
| Freshness SLA | How current the data is guaranteed to be and its update cadence |
| Quality contract | Defined expectations for completeness, accuracy, and consistency |
| Interfaces | How the product is consumed: Delta table, REST API, MCP tool, streaming feed |
| Discoverability | Catalogued and documented with schemas, examples, and stated use cases |

---

## How Data Products Are Consumed

A single data product can power all of the following:

- **Downstream data pipelines** — other data products compose more complex intelligence from simpler building blocks
- **Internal applications** — dashboards, risk tools, and research workflows query Gold views directly
- **Customer-facing APIs** — Gold views map cleanly to API responses, eliminating bespoke per-endpoint transformation
- **Agentic systems** — AI agents consume Gold-layer data products as structured, authoritative context for reasoning over the domain
- **MCP servers and tools** — data products exposed as MCP tools give AI assistants direct, scoped access to authoritative data from within assisted workflows

---

## In Practice: A Reference Data Product (illustrative)

A reference data product is the foundational intelligence layer for a diagnostic AI agent. The shape, with generic datasets:

| Dataset | Intelligence provided |
| --- | --- |
| Pricing | Current and historical pricing across vendors |
| Lifecycle | Lifecycle stage per product, from active to end-of-life |
| Availability risk | Discontinuation and supply-disruption signals |
| Substitutes | Validated substitutes with cross-reference metadata |

Each dataset is a modular Gold view, usable independently or composed with others. An agent can consume Lifecycle alone to diagnose risk, or join Pricing and Substitutes to recommend cost-effective replacements.



> The goal is not to ship data. It is to ship trusted intelligence, packaged for reuse: defined once, governed once, and consumable everywhere a decision is made — by a person, an application, or an agent.


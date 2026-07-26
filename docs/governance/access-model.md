# Access Model

Defines who can read and write each medallion layer, how access is requested, and how it is reviewed. This is the authoritative home of the access matrix.

## What this covers

- The role-by-layer access matrix
- The rules behind it
- How to request access and how requests are reviewed

## Access matrix

| Role | Bronze | Silver | Gold |
| --- | --- | --- | --- |
| Pipeline service principal | WRITE | WRITE | WRITE |
| Data engineers | READ | READ/WRITE | READ |
| Analysts / data scientists | none | READ (approved) | READ |
| Consuming services | none | none | READ |

## Rules

- Access is least privilege by default: requested per role and per layer, never granted broadly.
- Downstream consumers never get Bronze or Silver access. Gold is the only layer consuming services touch: analytics, GenAI retrieval, APIs, MCP servers. Any consumer connection to Bronze or Silver is a defect, not an exception.
- Analyst Silver access requires approval and is READ only. It exists for profiling and validation work, not for building consumer-facing outputs.
- Production writes go through pipeline service principals only. Human identities do not hold production WRITE.
- All grants live in Unity Catalog on groups, not individual users. Audit comes from UC system tables, not spreadsheets.
- Self-service means exploring certified Gold data. Direct business-user access to raw tables is not self-service; it is distributed data engineering without standards (see [BI practices guidance](../practices/bi-practices-guidance.md)).

## Requesting access

> **Status: stub.** The request and approval workflow is not yet documented.

To be written:

- Where to submit workspace and data access requests
- Approval roles and expected turnaround
- Access review cadence and revocation

## Sharp edges

- Granting an analyst Silver access "temporarily" for one report creates a permanent dependency. If a consumer needs a Silver measure, the fix is a Gold object, not a grant.
- Grants to individual users survive team changes silently. Group-based grants are the only auditable pattern.

## Checklist

- [ ] Every grant maps to a row in the access matrix
- [ ] No consuming service has any Bronze or Silver grant
- [ ] All grants are on groups, not users
- [ ] Production WRITE is held only by pipeline service principals

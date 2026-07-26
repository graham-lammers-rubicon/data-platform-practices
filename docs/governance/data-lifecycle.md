# Data Classification, Retention, and Deletion

> **Status: stub.** Scope and required decisions are agreed; the answers are not. This doc is not normative until the decisions below are made and the notice is removed.

Defines how data is classified, how long each layer keeps it, and how it is deleted when required. These policies must be designed against the architecture's standing facts: Bronze is append-only and is the replay source for Silver ([Medallion data practices](../practices/medallion-data-practices.md)), prod data is cloned into nonprod for testing ([Environments](../platform/environments.md)), and Delta time travel retains history beyond a DELETE.

## Decisions to make

Each requires an owner, a decision, and a date. Answers open.

| Decision | Question | Owner |
| --- | --- | --- |
| Regulatory scope | Does GDPR and/or CCPA apply? CCPA covers CA employee and B2B data above revenue thresholds; "no public users" is not the test. Legal determination, recorded here. | TBD (legal) |
| Classification scheme | Tiers (e.g., public / internal / confidential / PII) and the UC tag that carries the tier on tables and columns. | TBD |
| PII controls | Which classification tiers require row filters or column masks, and for which groups ([Access model](access-model.md)). | TBD |
| Nonprod data | Are prod clones permitted to carry PII, or is masking/synthetic data required before nonprod use? | TBD |
| Retention per layer | Landing volume retention (bounds Bronze full-refresh), Bronze retention (bounds SCD2 replay), Silver/Gold retention, log/audit retention. | TBD |
| Deletion path | If erasure is required: the procedure that reaches Bronze events, Silver SCD2 versions, Gold aggregates, nonprod clones, and Delta time travel (VACUUM policy). | TBD |
| Enforcement | How retention and deletion are executed and verified: jobs, predictive optimization settings, audit query. | TBD |

## Standing constraints the decisions must respect

- Bronze completeness is load-bearing: a retention window shorter than the longest replay need breaks the SCD2 rebuild promise. Retention is a contract with the pipelines, not just a cost setting.
- Deleting from an append-only event table is a design exception. If erasure is in scope, the exception must be designed (e.g., targeted delete plus VACUUM), not improvised per request.
- Clones drift: a deletion executed in prod does not propagate to existing nonprod clones. The deletion path must enumerate them.

## Checklist (activates when decisions land)

- [ ] Every table and sensitive column carries a classification tag
- [ ] PII tiers have row filters or column masks per the access model
- [ ] Retention windows declared per layer and enforced by automation
- [ ] Deletion procedure documented, tested, and covers clones and time travel
- [ ] Regulatory scope decision recorded with legal sign-off

## Sources

- To be added when decisions are made. Candidates: Unity Catalog row filters and column masks, Delta VACUUM and time travel retention documentation.

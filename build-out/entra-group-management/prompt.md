# Build Prompt: entra-group-management

Paste this prompt to a coding agent in an empty repository, alongside `hld.md` and the `samples/` folder. It builds V1 as scoped in the HLD.

---

You are building **entra-group-management**, a GitOps system that manages Microsoft Entra ID security groups from YAML. `hld.md` is the design of record; read it fully before writing code. `samples/` contains example group and policy files that must validate against your schema unchanged.

## Scope: V1 only

Per HLD section 9: schema, PR validation, and apply for the `groups/databricks/` folder, correlation-id stamping, audit records to Log Analytics, and the fixture-based regression suite. Do not build drift detection, issue-form intake, or downstream grant automation.

## Deliverables

1. **Repo scaffold** exactly as HLD section 4.1: `groups/`, `schema/`, `policy/`, `src/`, `tests/`, `.github/workflows/`. Copy `samples/groups/*` into `groups/` and `samples/policy/*` into `policy/` as the seed data.
2. **Pydantic models** (`src/models.py`): the group schema from HLD 4.2. Strict mode, unknown keys rejected, `owners` non-empty, `targets` an enum of `azure | azure-sql | databricks | github`, name validated against both patterns in `policy/naming.yaml`. Export JSON Schema to `schema/group.schema.json`.
3. **Reconciler CLI** (`src/`): `plan` and `apply` subcommands.
   - `plan`: read YAML + live Entra state (Graph, read-only), output the change set (creates, updates, deletes, member adds/removes) as JSON and as a markdown table for PR comments. Zero writes.
   - `apply`: execute the plan idempotently, nested groups first, deletes gated on the `deletions-approved` label. Every Graph write sends `client-request-id = uuid5(NAMESPACE, "<commit-sha>:<seq>")` and emits one `GroupsGitOps_CL` record (fields per HLD 4.4) to the Log Analytics data collection endpoint.
   - Policy enforcement in both: naming patterns, protected-groups denylist, `grp-` prefix fence, no role-assignable groups, member UPNs resolve in Entra, nested groups exist in repo.
4. **Workflows**: `pr-validate.yml` (schema + policy + plan, posts plan as PR comment, read-only SP) and `apply.yml` (push to main, environment-gated, write SP, concurrency group, no cancel-in-progress). OIDC via `azure/login`; no stored secrets.
5. **Tests** (`tests/`): the fixture eval set from HLD section 10. Fixture YAML sets with known-correct plans: create, delete, member add/remove, nested group ordering, invalid UPN, protected group rejection, naming violation, idempotent re-run producing an empty plan. Graph calls mocked; every fixture asserts the exact expected change set. These tests gate merge on the reconciler itself.
6. **README.md**: how to add a group (PR flow), how to run plan locally, how to run the tests. One page.

## Constraints

- Python 3.12+, `uv` for everything, Pydantic v2, `msgraph-sdk` with async I/O (`TaskGroup`, not `gather`).
- Non-defensive code: raise on failure, no catch-and-log, no silent fallbacks, no `.get()` where the key must exist. Trust the Pydantic models past the boundary.
- The apply must be safe to re-run at any point: partial failure followed by retry converges, never duplicates.
- No secrets in the repo or workflows. Auth is OIDC only. Tenant id, client ids, and the DCE endpoint are GitHub Actions variables.
- Functions over classes. If a file exceeds what one person reads in one sitting, it is too big.

## Acceptance criteria (HLD section 10)

- All sample files validate; a deliberately broken copy of each fails with a message naming the field and rule.
- `plan` against an unchanged tenant returns an empty change set.
- The fixture suite passes and runs in CI on every PR.
- A dry-run apply prints, for each would-be Graph write, the derived correlation GUID and the `GroupsGitOps_CL` record it would emit, so the correlation-id propagation test (HLD 4.6) can be executed against a real tenant as the first integration step.

## Verify before finishing

Run the full test suite, run `plan` against the seeded `groups/` with a mocked empty tenant (expect: creates for every group, nested first), and lint. Report what passes and what you could not verify without a real tenant.

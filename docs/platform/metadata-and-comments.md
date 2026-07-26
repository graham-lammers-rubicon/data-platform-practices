# Metadata and Comments

Names state what a thing is; this doc governs what it means. Comments, COMMENTs, and commit messages are the queryable record humans and agents use to understand the data landscape. [Naming conventions](naming-conventions.md) governs the names themselves.

## What this covers

- Unity Catalog COMMENT requirements per object type
- Git commit message standard
- Python comments and docstrings

## Unity Catalog COMMENTs

Every UC object carries a COMMENT. `COMMENT ON` supports catalogs, schemas, tables, columns, and volumes. An object without a COMMENT is incomplete.

| Object | COMMENT states |
| --- | --- |
| Catalog | Scope and owner |
| Schema | Domain and medallion layer |
| Table / view | Grain and content (see [Medallion data practices](../practices/medallion-data-practices.md)) |
| Column | Meaning, unit, semi-additive labeling where it applies |
| Volume | Purpose |

Rules:

- COMMENTs are queryable: `INFORMATION_SCHEMA.TABLES.COMMENT`, `INFORMATION_SCHEMA.COLUMNS.COMMENT`. Humans and agents discover the landscape through them, not tribal knowledge; an uncommented object is invisible to both.
- COMMENTs live in source: DDL and pipeline definitions in repos, deployed with the object. A COMMENT set ad hoc in the workspace is not in source control and drifts from the deployed definition.

## Commit messages

The Git project's convention: capitalized imperative subject of about 50 characters, blank line, body wrapped at about 72 characters explaining motivation and how the change contrasts with previous behavior.

History is how humans and agents reconstruct intent; `fix`, `wip`, and `updates` destroy it.

## Python comments and docstrings

- Docstrings on all public modules, functions, classes, and methods (PEP 257).
- Comments are complete sentences, kept current with the code, and explain what the code cannot say; inline comments sparingly, never restating the line (PEP 8).

## Sharp edges

- Comments that contradict the code are worse than no comments (PEP 8). The update rule is absolute: change the code, change the comment.
- A Gold object without column COMMENTs degrades every consumer that reads metadata: catalogs, BI tools, GenAI retrieval. The cost lands downstream, not on the author.
- Commit subjects written past tense or as topic labels break the convention shared by `git merge` and `git revert`; the history stops reading as a sequence of commands.

## Checklist

- [ ] Every UC object has a COMMENT with the required content for its type
- [ ] COMMENTs are defined in source, not set ad hoc in the workspace
- [ ] Commit subjects are capitalized, imperative, about 50 characters; bodies explain why
- [ ] All public Python modules, functions, classes, and methods have docstrings

## Sources

- Databricks: [COMMENT ON](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-syntax-ddl-comment)
- Databricks: [INFORMATION_SCHEMA.COLUMNS](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/information-schema/columns)
- Pro Git: [Commit guidelines](https://git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project#_commit_guidelines)
- Python: [PEP 8, Comments](https://peps.python.org/pep-0008/#comments), [PEP 257, Docstring Conventions](https://peps.python.org/pep-0257/)

---
name: jbhifi-customized-skills
description: "Router for reviewing a Power BI report built from a JBHIFI Group reporting template. Load this whenever asked to review, audit, or check the standard of a JBHIFI-based report or semantic model, then load the matching per-template doc it points to."
---

> **CRITICAL NOTE — templates can be wrong.** The per-template docs below document what each
> template's live model actually does, not an assumed-correct ideal. Some documented findings are
> "critical, blocking" precisely because the template itself has a defect (e.g. the GRP template's
> confirmed `SemanticError` measures) — don't treat every observed pattern as house style to
> imitate. Separate "this is the convention, match it" from "this exists today but is a bug" per
> finding, the way the two docs already do.

# JBHIFI Report Review — Router

This folder holds one review-standard doc per JBHIFI reporting template, each written from a live
MCP inspection of that template's actual semantic model (not generic best practice). Pick the
right doc before reviewing a report — the two templates have diverged (see each doc's own
divergence notes), so don't apply one template's doc to a report built from the other.

## Which doc to load

| Report was built from... | Load |
|---|---|
| `JB Hi-Fi Reporting Template` | [jb-hifi-reporting-template-review-skill.md](./jb-hifi-reporting-template-review-skill.md) |
| `GRP Supply Chain Reporting Template` | [grp-supply-chain-reporting-template-review-skill.md](./grp-supply-chain-reporting-template-review-skill.md) |
| Something else / unsure which template it descends from | See below, then write a new sibling doc |

## If you don't know which template a report descends from

Connect to the report's model (per `semantic-model-authoring`'s
[Tool Selection Priority](../semantic-model-authoring/SKILL.md#tool-selection-priority)) and check
for tables unique to each template — this is more reliable than lineage tags, since the shared
scaffolding (`Metrics` table, `Sales Dollars` measure, etc.) has identical `lineageTag`s across
both templates:

- Has `Country`, `Selling Channel`, `Promotions`, or `Time Group` tables → **JB Hi-Fi** lineage.
- Has `Line of Business`, `Supplier`, `Product V2`, `Group Employee`, `Group Product`, or
  `Returns App Statuses` tables → **GRP Supply Chain** lineage.
- Has neither set, or a mix that doesn't match either → treat it as a new/unknown template. Don't
  force-fit one of the two existing docs. Inspect it fresh via MCP, write a new sibling doc in
  this folder following the same structure (shared-scaffolding note, per-section findings graded
  critical/recommended/optional, a "Findings format" closer), and add a row to the table above.

## How to run the review itself

Both per-template docs assume the `semantic-model-authoring` skill's
[Analyze Best Practices](../semantic-model-authoring/SKILL.md#workflow-analyze-best-practices)
workflow for the generic checks (star schema, relationship cardinality, explicit measures,
`formatString`, hidden FK columns) — connect via MCP first, then layer the JBHIFI-specific and
per-template checks from the matched doc on top. Present findings grouped by
**critical / recommended / optional**, citing the section of the matched doc, and wait for
approval before applying any fix.

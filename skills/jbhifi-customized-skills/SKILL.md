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

## Review philosophy — standard compliance, not exact replication

**Every report will legitimately have different measures and DAX than the template — that's
expected, not a defect.** Don't flag a report for having a measure the template doesn't, lacking
one the template has, or computing something differently. Judge it against the *standard* the
template's own patterns demonstrate instead:

- **Naming convention** is followed (technical keys hidden + `ALL_CAPS_SNAKE_CASE`, business
  columns visible + `Title Case`) — regardless of what those columns actually represent.
- **Formula/DAX standard** is followed (explicit `formatString`, `ISINSCOPE` guards on comparative
  measures, no dead or broken references, no `SemanticError`/`DependencyError` states) —
  regardless of what the formula calculates.
- **Relationships are up to standard** (sensible cardinality, any bidirectional/many-to-many path
  justified, keys correctly hidden) — not that the relationship *list* matches the template's.
  This includes the fact table itself: **every fact table must have active relationships to every
  dimension it needs to be filtered by, whatever that fact table is actually named.** The
  per-template docs illustrate this using their own fact table's name (`Sales`, in both templates
  today) because that's what these two templates happen to call it — don't search for a table
  literally named "Sales" in a report that isn't derived from these templates, or in a future
  template variant with a differently-named fact table (e.g. a customer-traffic or returns
  report). Find the report's actual fact table and check that.
- **No unused measures** — every measure should be referenced by something (a visual, another
  measure, RLS logic). One that isn't is a defect regardless of whether it matches a template
  pattern. Full confirmation of "referenced by a visual" needs the report side
  (`powerbi-report-authoring` / `powerbi-report-management`, PBIR inspection) — the semantic-model
  MCP alone can only confirm measure-to-measure references, not visual usage. Flag candidates from
  the MCP-only pass, but don't declare a measure dead without checking the report pages too.

**Company guidance now covered**: each per-template doc's last content section (before "Findings
format") documents JBHIFI Group-wide standards for native-visual preference, the JB colour
palette (with hex codes), and report/model performance & capacity guidance (Performance Analyzer
workflow, avoiding Matrix visuals, complexity, granularity). These apply to any report regardless
of template.

**Still out of scope here — don't let that mean "skipped"**: actually auditing a report's visuals
— which specific visual type is used on which page, actual font sizes, slicer/visual pixel
placement, whether the colour palette above was *actually applied* rather than just documented as
a rule — requires reading the report's PBIR/pages, which none of these docs do (they're written
purely from `powerbi-modeling-stdio` MCP inspection of the semantic model). A complete report
review needs a separate pass with `powerbi-report-design` (visual/theme critique) and
`powerbi-report-authoring` (PBIR mechanics) — say so explicitly when handing back a
semantic-model-only review, rather than implying the visuals themselves were checked.

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

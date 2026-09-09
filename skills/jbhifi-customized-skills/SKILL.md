---
name: jbhifi-customized-skills
description: "Router for reviewing a Power BI report built from a JBHIFI Group reporting template. Load this whenever asked to review, audit, or check the standard of a JBHIFI-based report or semantic model, then load the matching per-template doc it points to."
---

> **CRITICAL NOTE — templates can be wrong.** The per-template docs below document what each
> template's live model actually does, not an assumed-correct ideal. Some documented findings are
> "critical, blocking" precisely because the template itself has a defect (e.g. the GRP template's
> confirmed `SemanticError` measures, or the TGG template's confirmed `Dlivered` typo and
> `Report Refreshed Text` name/formula mismatch) — don't treat every observed pattern as house
> style to imitate. Separate "this is the convention, match it" from "this exists today but is a
> bug" per finding, the way the per-template docs already do.

# JBHIFI Report Review — Router

This folder holds one review-standard doc per JBHIFI reporting template, each written from a live
MCP inspection of that template's actual semantic model (not generic best practice). Pick the
right doc before reviewing a report — the templates are architecturally distinct (JB Hi-Fi and GRP
share a common ancestor and have diverged; TGG is a structurally unrelated third family built on
`EXTERNALMEASURE`/DirectQuery federation) — so don't apply one template's doc to a report built
from another.

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
- **Measure names must match their formulas** — a measure's name is a promise about what it
  computes; if the DAX and the name disagree, that's a defect independent of whether the DAX
  itself is error-free. This is a manual/semantic check (no `state` field flags it) — read each
  measure's expression and ask whether its name unambiguously describes it, including partial
  mismatches (a name that's technically true but omits a scope qualifier the model actually
  needs, e.g. a generic name on a measure that only covers one of several product lines).
- **No unused measures, tables, or other assets** — every measure should be referenced by
  something (a visual, another measure, RLS logic); the same logic extends to whole tables (a
  dimension with no relationships and no visual reference is dead weight, not just an unused
  measure) and to non-measure assets — imported custom visuals never placed on a page, unused
  bookmarks, RLS roles nothing is assigned to, Field Parameter rows nothing references, orphaned
  static resources (themes, images). One that isn't used is a defect regardless of whether it
  matches a template pattern. Full confirmation of "referenced by a visual/page" needs the report
  side (`powerbi-report-authoring` / `powerbi-report-management`, PBIR inspection) — the
  semantic-model MCP alone can only confirm measure-to-measure references and a table's
  relationship count, not visual or asset usage. Flag candidates from the MCP-only pass, but don't
  declare something dead without checking the report pages too.

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
| `GRP Reporting Template` | [grp-reporting-template-review-skill.md](./grp-reporting-template-review-skill.md) |
| `TGG PBI Style Template` (or any report federating measures via `EXTERNALMEASURE`/DirectQuery to a shared AS dataset) | [tgg-reporting-template-review-skill.md](./tgg-reporting-template-review-skill.md) |
| Something else / unsure which template it descends from | See below, then write a new sibling doc |

> The `.pbip` file this row refers to may still literally be named `GRP Supply Chain Reporting
> Template` on disk — this table's left column is descriptive, not a literal filename match.
> Don't route by comparing an exact name string; use the table-presence check below, which is
> reliable regardless of what the file is actually called.

## Ask first — don't guess lineage if you can just ask

**Before inspecting anything, ask the user which template/report family this is** (JB Hi-Fi, GRP,
TGG, or name the actual report) — it's one question and it's authoritative, whereas every
detection method below is a heuristic that can be wrong. Only fall back to table-presence
detection when the user doesn't know or hasn't said.

## If you don't know which template a report descends from

Connect to the report's model (per `semantic-model-authoring`'s
[Tool Selection Priority](../semantic-model-authoring/SKILL.md#tool-selection-priority)) and check
for tables/patterns unique to each template — more reliable than lineage tags, since JB Hi-Fi and
GRP's shared scaffolding (`Metrics` table, `Sales Dollars` measure, etc.) has identical
`lineageTag`s across both:

- **Almost every measure is `EXTERNALMEASURE(...)`, and/or `PBI_QueryOrder`/a table's
  `expressionSourceName` shows a DirectQuery source like `"DirectQuery to AS - <name> Dataset"`**
  → **TGG** lineage. Check this signal *first* — it's structural (Import vs. federated DirectQuery),
  not a table-name pattern, and is the most reliable of the three since it can't coincidentally
  overlap with the other two templates' Import-mode, locally-authored-DAX design.
- Has a `Line of Business` table, or **any table whose name starts with or contains `Group`,
  `GROUP`, or `GRP`** (e.g. `Group Employee`, `Group Product`, a future `GRP ...`/`Group ...`
  table not yet seen) → **GRP** lineage. Treat this as a naming-pattern match, not a fixed list —
  the specific tables in §5/§6 of the GRP doc are *examples* found in this template, not the
  complete set a future GRP-derived report must have.
- Has `Country`, `Selling Channel`, `Promotions`, or `Time Group` tables → **JB Hi-Fi** lineage.
  A weaker, secondary signal: JB Hi-Fi-lineage reports are often scoped to a single country (data
  filtered to `Country = 1` / `"AU"` in the `Country`/RLS tables) — don't rely on this alone, it's
  corroborating evidence at best, and check it against the dimension tables actually present, not
  a specific hardcoded value.
- Matches none of the above, or a mix that doesn't fit any → treat it as a new/unknown template.
  Don't force-fit one of the existing docs. Inspect it fresh via MCP, write a new sibling doc in
  this folder following the same structure (shared-scaffolding note, per-section findings graded
  critical/recommended/optional, a "Findings format" closer), and add a row to the table above —
  this is exactly how the TGG doc came to exist.

## How to run the review itself

All per-template docs assume the `semantic-model-authoring` skill's
[Analyze Best Practices](../semantic-model-authoring/SKILL.md#workflow-analyze-best-practices)
workflow for the generic checks (star schema, relationship cardinality, explicit measures,
`formatString`, hidden FK columns) — connect via MCP first, then layer the JBHIFI-specific and
per-template checks from the matched doc on top. Present findings grouped by
**critical / recommended / optional**, citing the section of the matched doc, and wait for
approval before applying any fix.

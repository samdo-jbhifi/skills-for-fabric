---
name: jb-hifi-reporting-template-review-skill
description: "Review standard for Power BI reports built from the JB Hi-Fi Reporting Template. Load this before reviewing, auditing, or approving a report derived from that template, to check it against the template's actual conventions rather than generic best practices alone."
---

# JB Hi-Fi Reporting Template — Report Review Standard

This checklist is derived from a live inspection (via the `powerbi-modeling-stdio` MCP server,
connected to Power BI Desktop) of the **JB Hi-Fi Reporting Template** semantic model on
2026-09-04. It documents the conventions that template actually implements, so a review can
check a derived report against *this template's own standard*, not just generic Power BI best
practice.

> **Sibling template**: `GRP Supply Chain Reporting Template` (same `JBHIFI_GROUP_TEMPLATES/`
> folder) shares this template's core scaffolding — confirmed via matching `lineageTag`s on the
> `Metrics` table and `Sales Dollars` measure, i.e. same common ancestor — but has since diverged
> in ways that matter for review. See
> [grp-supply-chain-reporting-template-review-skill.md](./grp-supply-chain-reporting-template-review-skill.md)
> for the GRP-specific deltas (including a confirmed broken measure) rather than assuming the two
> templates are identical today.

## How to run this review

1. Connect to the target model via `connection_operations` (`ListLocalInstances` → `Connect`), per
   the `semantic-model-authoring` skill's [Tool Selection Priority](../semantic-model-authoring/SKILL.md#tool-selection-priority).
   MCP is Tier 1 — when connected, treat the live model as the source of truth and do **not**
   read `.tmdl` files directly.
2. Follow the `semantic-model-authoring` skill's [Analyze Best Practices](../semantic-model-authoring/SKILL.md#workflow-analyze-best-practices)
   workflow for the generic checks (star schema, relationship cardinality, explicit measures,
   `formatString`, hidden FK columns). Apply the JBHIFI-specific checks below on top of it.
3. Present findings grouped by severity (critical / recommended / optional), as that workflow
   does, and wait for approval before applying fixes.

## 1. Metadata measures contract (`00. Metadata` folder)

Every JBHIFI report ships six placeholder measures in the `Metrics` table, folder `00. Metadata`:
`Report Name`, `Report Footer Text`, `Report Tooltip`, `Most Recent Data Refresh`,
`Line of Business Colour`, `Line of Business Font Colour`.

- **Critical**: `Report Name` must not still read the template placeholder
  `"Report Name (MEASURE)"` (its own DAX comment says `-- Ensure this is the same name as the
  report`). Same for `Report Footer Text` / `Report Tooltip` if they carry similar placeholders.
- **Recommended**: `Line of Business Colour` / `Line of Business Font Colour` use a
  `SWITCH(MIN('Country'[COUNTRY_ID]), 1, ..., 2, ..., ...)` pattern keyed off country. In the
  template every branch returns the same value (`"#fff200"` for both JBAU and JBNZ) — that's the
  unconfigured default. If a report is genuinely multi-country/multi-line-of-business, confirm the
  branches were actually differentiated, not left identical.
- **Optional**: `Most Recent Data Refresh` formats with `FORMAT(..., "DD/MM/YYYY", "en-US")` — the
  locale argument (`en-US`) is inconsistent with the model's own culture (`en-AU`). Literal format
  tokens make this harmless today, but prefer `"en-AU"` for consistency if touched.

## 2. Measure-table pattern

All measures live in one `Metrics` table (columns hidden, only measures exposed), organized into
numbered display folders: `00. Metadata`, `01. Base Measures`, `02. Comparative Measures`. When a
report adds new measures:

- **Critical**: new measures go into `Metrics` (or a clearly-named sibling measure table), not
  scattered across dimension/fact tables.
- **Recommended**: extend the numbered-folder convention (e.g. `03. <New Category>`) rather than
  inventing an unnumbered folder name — the numbering is what keeps folder order stable in the
  field list.

## 3. Explicit `formatString` on every measure

The template itself is inconsistent here: `Sales Dollars`, `Sales Quantity`, and every
`YoY Var %` / `LY` / `YTD` measure inspected had an **empty** `formatString`. This is a gap, not a
standard to imitate.

- **Recommended**: every measure should carry an explicit `formatString` — currency measures
  (`Sales Dollars`, `Sales Dollars LY`, `Sales Dollars YTD`, …) and percentage measures
  (`... Var %`) especially. Flag any measure reviewed with `formatString: ""` and propose one
  (e.g. `$#,0` for dollar measures, `0.0%` for variance-percent measures) rather than leaving
  formatting to the report canvas.

## 4. Naming convention: technical keys vs. business columns

Verified via `column_operations Get`:

- Technical/surrogate key columns keep source-system `ALL_CAPS_SNAKE_CASE` naming
  (`COUNTRY_STORE_ID`, `DATE_KEY`, `TIME_ID`, `SUB_SALE_CHANNEL_ID`, …) **and are `isHidden: true`**.
- Business-facing columns use Title Case with spaces (`Store Name`, `Sale Price`,
  `Written Sales Date`) and **are visible** (`isHidden: false`).

Review rule:
- **Critical**: any visible column still in `ALL_CAPS_SNAKE_CASE` — it will leak a technical key
  into a report's field list. Either hide it or confirm it's genuinely business-facing and rename it.
- **Critical**: any hidden column that is actually a business attribute a report author needs —
  it should be un-hidden and (if snake_case) renamed to Title Case.

## 5. Retail fiscal calendar (`Date` table, 109 columns)

The `Date` table carries a full retail (52/53-week) fiscal calendar **and** a parallel "Pnl"
(P&L) calendar, plus precomputed offsets:

- Fiscal hierarchy columns: `Fiscal Week/Month/Quarter/Half/Year Code`, `... Name`,
  `... Start Date`/`End Date`, `Day Of Fiscal ...`.
- P&L hierarchy: `Pnl Year/Quarter/Period Code`, `Pnl Calendar Year/Month Code`, offset variants
  (`Pnl Offset ...`).
- Last-year / last-last-year offsets are **precomputed date columns**, not DAX time intelligence:
  `Ly Day`, `Ly Week/Month/Quarter/Year Start/End Date`, `Lly Day` (and `Pnl Ly Day` / `Pnl Lly Day`
  equivalents).
- Boolean flags: `Is Yesterday`, `Is Wtd`, `Is Mtd`, `Is Ytd`, `Is Rolling 7`, `Is Rolling 28`,
  `Is Pnl Mth`, `Is Pnl Year`.

The model has `__PBI_TimeIntelligenceEnabled: "0"` (native auto date/time is off, correctly, since
an explicit Date dimension exists) — every LY/comparative measure inspected
(`Sales Dollars LY`, `Sales Dollars YoY Var %`, …) uses the pattern:

```dax
SWITCH(TRUE,
    ISINSCOPE('Date'[DATE_KEY]), CALCULATE([Base Measure], USERELATIONSHIP('Sales'[Written Sales Date], 'Date'[Ly Day])),
    BLANK()
)
```

**This pattern only works while `Sales` has a relationship (active or inactive) whose two columns
are exactly `Sales[Written Sales Date]` and `Date[Ly Day]`.** See the GRP sibling doc for what
happens when that relationship is missing — `USERELATIONSHIP` throws a semantic error, not a
silent blank.

Review rule:
- **Critical**: `__PBI_TimeIntelligenceEnabled` must stay `"0"`. If a report re-enables it (e.g.
  via a new date column with auto date/time), that conflicts with the explicit `Date` table and
  will create duplicate/ambiguous date hierarchies.
- **Critical**: any new LY/LLY/YoY-style measure must follow the `ISINSCOPE` guard +
  `USERELATIONSHIP` pattern against the precomputed offset columns — not
  `SAMEPERIODLASTYEAR`/`DATEADD`, which assume a standard Gregorian calendar and will misalign
  against the retail fiscal calendar.
- **Critical**: before relying on any `USERELATIONSHIP`-based measure, confirm the relationship it
  names actually exists in `relationship_operations List` — don't just trust the DAX compiles at
  a glance; check the measure's `state` field for `SemanticError`.

## 6. Relationships — bidirectional / many-to-many paths

Ten relationships were inspected. Three use `BothDirections` cross-filtering, two of which are
many-to-many:

| From → To | Cardinality | Cross-filter |
|---|---|---|
| `Date Group` → `Date` | Many:One | BothDirections |
| `Sales` → `Employee` (`COUNTRY_EMPLOYEE_CODE`) | Many:Many | BothDirections |
| `Sales` → `Promotions` (`COUNTRY_PROMOTION_ID`) | Many:Many | BothDirections |

Plus one **inactive** relationship (`Sales[Written Sales Date]` → `Date[Ly Day]`), activated only
via `USERELATIONSHIP` inside LY measures (per §5) — that's the correct, intentional pattern.

Review rule:
- **Recommended**: any *new* bidirectional or many-to-many relationship should be justified in
  the relationship's description or PR notes — stacking bidirectional paths on top of the existing
  three risks ambiguous filter propagation. Don't add one without checking what it does to
  existing cross-filter behavior on `Sales`.
- **Optional**: if a new inactive relationship is added for another comparison (e.g. 2-years-ago),
  follow the same "inactive + `USERELATIONSHIP` inside a guarded measure" pattern rather than a
  second active relationship.

## 7. Row-level security table (`PBI User Security`)

A hidden table (`isHidden: true`), sourced from a Power Platform dataflow, columns:
`COUNTRY_ID`, `STORE_ID`, `COUNTRY_STORE_ID`, `AREA_ID`, `REGIONAL_ID`, `AREA_MANAGER_ID`,
`Store Email`, `Area Manager Email`. Its M query comment: *"This table is especially important
when limiting data depending on the user via Row-Level Security (RLS)."*

Review rule:
- **Critical**: this table must remain hidden in any derived report. If a report author needs new
  RLS scopes (e.g. by regional manager), extend this table's source query rather than building a
  parallel security table.
- Per the `semantic-model-authoring` skill's DENY rule: do **not** attempt to add/remove RLS role
  *membership* as part of this review — flag membership concerns and redirect to the Power BI
  portal; only the table's shape (columns, hidden state) is in scope here.

## 8. Template scaffolding — Field Parameters, Kada, connection parameters

- **Field Parameter tables** — `Metrics (FP)` and `Dimensions (FP)` use the `NAMEOF(...)` field
  parameter pattern to drive dynamic-axis visuals. `Metrics (FP)` additionally carries a `Units`
  column (`Quantity`/`Dollars`) for slicing by metric unit — a JBHIFI-specific extension of the
  stock Field Parameter pattern. New metrics/dimensions a report exposes for dynamic switching
  should be added as rows here, following the same 4-tuple shape
  (`name, NAMEOF(...), order, unit`).
- **Kada integration** — `Kada Widget` (hidden) sources from a `Kada Param` query, feeding a data
  governance/catalog widget. Don't delete or rename without confirming the Kada widget isn't
  relying on it.
- **Connection parameterization** — the model's `PBI_QueryOrder` annotation references
  `Datasource`, `Warehouse`, `Database` queries (Snowflake connection parameters) that are *not*
  loaded as model tables — that's correct; they're M parameters, not data tables. Don't "fix" this
  by loading them into the model.
- **Preserve authoring comments** — every one of these scaffold objects ships with an inline M/DAX
  comment explaining its purpose and an "Authored: <month year>" tag. When a report customizes one
  of these, update the comment rather than deleting it — it's the only in-model documentation a
  future maintainer gets.

## 9. Model-level settings to hold constant

From `model_operations Get` on the template: `defaultMode: Import`, `culture: en-AU`,
`compatibilityLevel: 1606`, `__PBI_TimeIntelligenceEnabled: "0"`.

- **Critical**: `culture` should stay `en-AU` for JBHIFI reports (affects default number/date
  formatting) unless the report is explicitly for a different region.
- **Optional, not template-conformant today**: `discourageImplicitMeasures` is `false` in the
  template. Tightening this to `true` (forcing explicit measures only) is generally good practice,
  but doing so is a deliberate model-wide change, not something to flip silently during a report
  review — raise it as a suggestion, don't apply it as part of a routine review.

## Findings format

Report findings the same way the `semantic-model-authoring` "Analyze Best Practices" workflow
does — grouped by **critical / recommended / optional**, each stating the rule violated (cite the
section above) and the proposed fix — and wait for user approval before applying any fix.

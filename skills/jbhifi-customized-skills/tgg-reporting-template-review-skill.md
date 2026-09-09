---
name: tgg-reporting-template-review-skill
description: "Review standard for Power BI reports built from the TGG PBI Style Template. Load this before reviewing, auditing, or approving a report derived from that template. This template is architecturally unrelated to the JB Hi-Fi/GRP templates — almost all its measures are EXTERNALMEASURE passthroughs to a shared upstream Analysis Services dataset, which changes what a local-file review can and can't verify."
---

# TGG PBI Style Template — Report Review Standard

This checklist is derived from a live inspection (via the `powerbi-modeling-stdio` MCP server,
connected to Power BI Desktop) of the **TGG PBI Style Template v1.6** semantic model on
2026-09-09.

> **Not a JB Hi-Fi/GRP sibling.** This template shares no lineage with
> [jb-hifi-reporting-template-review-skill.md](./jb-hifi-reporting-template-review-skill.md) or
> [grp-reporting-template-review-skill.md](./grp-reporting-template-review-skill.md) — no `Metrics`
> table, no `Kada Widget`, no `PBI User Security`, culture is `en-US` (not `en-AU`),
> `__PBI_TimeIntelligenceEnabled` is `"1"` (enabled, not disabled), and it's **DirectQuery**
> against another Analysis Services dataset, not Import. Don't apply the other two docs' checks
> here — this is its own template family with its own conventions, documented below.

## 0. Critical — this file has almost no local DAX to review; the real logic lives upstream

**The single most important thing to understand before reviewing this template**: of the 288
measures inspected, all but one (`Report Refreshed Text`) are `EXTERNALMEASURE(...)` wrappers:

```dax
EXTERNALMEASURE("Delivered Sales$", CURRENCY, "DirectQuery to AS - Sales Dataset")
```

This is Power BI's measure-federation feature — the local model doesn't compute anything itself;
it re-exposes measures already defined in a shared upstream Analysis Services model (registered
here as the DirectQuery source `"DirectQuery to AS - Sales Dataset"`). Even `Version History`
(the dev changelog table) is itself a DirectQuery entity from that same upstream source, not a
local table.

**Review rule**:
- **Critical**: don't attempt to review DAX *correctness* for an `EXTERNALMEASURE`-wrapped measure
  from this file alone — the actual expression isn't visible here, only its name, type, and
  `formatString`. Checks like "does this `ISINSCOPE` guard work," "is `USERELATIONSHIP` valid," or
  "does the formula match the name" (§ below) require connecting to the actual upstream
  `Sales Dataset` model, not this template.
- **Critical**: colour/icon-coding measures (`Delivered Sales Icon`, `Delivered Sales Font Color`,
  etc. — see §5) are also externalized. Verifying their actual hex/icon output against the JB
  colour palette (§6) is not possible from this file; it requires the upstream model.
- **Recommended**: when reviewing a report built from this template, identify and get access to
  the upstream `Sales Dataset` model early — most of what would normally be "read the DAX and
  check it" for a JB Hi-Fi/GRP-style report is a no-op here without it.

## 1. Measure-table-per-domain pattern (this template's version of the measure-table convention)

Unlike JB Hi-Fi/GRP's single `Metrics` table, this template splits measures across **six
domain-named tables**, each following the same shape (single hidden `RowNumber` column,
`mode: DirectQuery`, sourced as a DirectQuery entity from the same upstream dataset):
`Sales Measures` (119), `Budget Measures` (18), `Profitability Measures` (50),
`Rebate Measures` (53), `Employee Cost Measures` (25), `Utility Measures` (22).

Review rule:
- **Recommended**: any new measure should go into the domain table matching its topic (or a new,
  clearly-named `<Domain> Measures` table) — don't scatter measures across dimension tables, and
  don't invent a differently-named catch-all table when an existing domain table fits.
- **Recommended**: each measure also carries a `displayFolder` sub-grouping within its table
  (`Sales Metric`, `LY Metric`, `LLY Metric`, `PnL LY Metric`, `Date Range Comparison Metric`,
  `Discount Metric`, `Utility`, `Context Text`, `Selected`, `Button Text`) — follow this same
  sub-folder convention for new measures rather than leaving `displayFolder` blank.

## 2. Naming convention: `$`/`%` suffix style, and a confirmed typo

This template's own naming convention differs from JB Hi-Fi/GRP's (which spell out
"Dollars"/"Var %") — here, measures append `$`/`%` directly to the name:
`Delivered Sales$`, `LY Delivered Sales Var%`, `Written Discount% Var`. Neither style is "more
correct" — the standard to hold is *internal consistency*, not matching the other templates.

**Confirmed defect**: `PnL LY Dlivered Discount% Var` and `PnL LLY Dlivered Discount% Var` both
misspell "Delivered" as "Dlivered" — a real typo, not a naming-style choice, verified directly
from the measure list.

Technical join-key columns are correctly hidden and self-documented: `STORE_ID`, `PROD_ID`,
`CHANNEL_SOURCE_ID`, `DAY` in `Delivered Sales` all carry `isHidden: true` **and** a column
`description` explicitly stating `"...should be hidden"` — confirmed via `column_operations Get`
on `STORE_ID`.

Review rule:
- **Critical**: fix confirmed spelling defects like `Dlivered` — these aren't style, they're typos
  that will confuse anyone searching the field list for "Delivered."
- **Recommended**: hold the `$`/`%` suffix convention consistently for any new measure in this
  template — don't mix in spelled-out "Dollars"/"Percent" naming from a different template family.
- **Recommended**: when a column's `description` says it "should be hidden," verify `isHidden` is
  actually `true` — a description making a claim the metadata doesn't back up is the same
  defect class as §9's "measure name must match its formula," just applied to columns.

## 3. Column-level display folders on wide dimension tables

`Store` (21 columns) organizes its columns into `Attributes`, `Dates`, and `Physical Location`
display folders — a pattern JB Hi-Fi/GRP don't use (they only organize at the table/measure
level, never per-column on a dimension).

Review rule:
- **Recommended**: for any wide dimension table (start considering above ~15-20 columns), use
  column-level `displayFolder`s to group related attributes — follow `Store`'s folder names where
  the concept matches (`Dates`, `Physical Location`, `Attributes`) rather than inventing new ones
  for the same kind of grouping.

## 4. Triple calendar system, plus a user-selectable comparison range

The `Date` table (119 columns) supports three calendar systems in parallel: TGG's retail
(fiscal) calendar, a PnL calendar, and the Gregorian calendar — an even larger version of JB
Hi-Fi's dual Fiscal+PnL pattern (109 columns).

**Novel pattern not seen in JB Hi-Fi/GRP**: a separate `Date (comparison)` table (20 columns)
mirrors every fiscal/PnL/calendar column from `Date`, suffixed `(comparison)` — e.g.
`Fiscal Week Name (comparison)`, `PnL Period Name (comparison)`. It's joined via an **inactive**
`Date[Day]` ↔ `Date (comparison)[Comparison Day]` relationship (`BothDirections`,
`One:One`). This lets a report user pick an arbitrary comparison date range (not just fixed
LY/LLY), activated presumably via `USERELATIONSHIP` in the `Comparison ...` measures (§1) — though
per §0, that DAX lives upstream and can't be confirmed from this file.

Review rule:
- **Recommended**: if a new report needs an ad-hoc (not fixed-offset) comparison period, follow
  this `<Table> (comparison)` naming and inactive-relationship pattern rather than building a
  parallel mechanism.
- **Critical** (per §0): before trusting any `Comparison ...` measure, confirm with the upstream
  model owner that its `USERELATIONSHIP` target actually matches this inactive relationship — the
  local file can't verify this itself.

## 5. Grain-guard measures for budget rollups

`Is Delivered Budget in Scope?` / `Is Written Budget in Scope?` are boolean measures whose
description explains their purpose precisely: budgets are only defined at a specific grain
(Store/Day/Product Type/Selling Code for Delivered; Store/Day/Selling Code for Written), and these
booleans prevent budget measures from displaying misleading values when a visual is rolled up
above or below that grain.

Review rule:
- **Recommended**: this is a good, generalizable pattern for *any* measure defined at a fixed
  grain (budgets, targets, snapshots) — wrap its display with an `Is ... in Scope?` guard rather
  than letting it silently show a wrong (over/under-aggregated) value at the wrong rollup level.
  Compare to JB Hi-Fi/GRP's `ISINSCOPE('Date'[DATE_KEY])` guard on LY measures (JB Hi-Fi doc §5) —
  same defensive principle, applied to grain instead of time-context.

## 6. Row-level security: none defined locally — confirm, don't assume

`security_role_operations List` returns **zero roles** on this model. Unlike JB Hi-Fi/GRP (both of
which have a `PBI User Security` table driving RLS), this template has no local RLS at all.

Review rule:
- **Recommended, not automatically a defect**: given this model is DirectQuery to a shared
  upstream dataset (§0), row-level security is plausibly enforced there instead of locally — that
  would be a reasonable federated-model design. But **don't assume that without checking** — confirm
  with the model owner whether the upstream `Sales Dataset` model enforces RLS, and if a report
  built from this template needs per-user/per-store restriction, verify it's actually applied
  before shipping, rather than trusting that "it must be handled somewhere upstream."

## 7. Any "data freshness" measure must use an actual data timestamp, not wall-clock time

**General standard (applies to any report, whatever such a measure is actually named)**: a
measure whose label claims to show when data was refreshed/updated/"as at" a certain time must
compute that from something data-driven — a `MAX()` over an actual fact-table date column, or a
true refresh-metadata timestamp — not from `NOW()`/`TODAY()`. A viewer reading "data refreshed as
at ..." reasonably assumes it reflects the data, not the moment they happened to open the report.

**How to find this in a report you're reviewing**: don't search for a measure literally named
`Report Refreshed Text` — that's just what this template happens to call it, and a future report
(including one from this same template family) may name its equivalent something else, or have
none at all. Instead search measure names/descriptions for freshness-related language ("refresh",
"updated", "as at", "last data", "current as of") and check *those* measures' formulas — and if a
report genuinely has no such measure, there's nothing to flag here, not a gap to invent.

**Evidence found in this template**: its one non-externalized measure,
`Report Refreshed Text` —

```dax
"Data refreshed as at" & FORMAT(NOW()+(10/24),"dd/mm/yy h:mm AM/PM")
```

— shows the **current wall-clock time** (`NOW()`, offset by a hardcoded `+10` hours) at the
moment the report is viewed, not when the underlying data was actually last refreshed. Compare to
JB Hi-Fi's `Most Recent Data Refresh`, which computes `MAX('Sales'[Written Sales Date])` — an
actual data-driven timestamp.

Review rule:
- **Critical**: any measure making a data-freshness claim that a `NOW()`/`TODAY()`-based formula
  doesn't support — this template's own `Report Refreshed Text` is a confirmed instance, and it's
  a direct case of §9's "measure name must match its formula" check.
- **Recommended**: if actual data-refresh currency matters for a report, use the true last-refresh
  timestamp from the upstream source (if exposed) or the DirectQuery connection's own refresh
  metadata — not `NOW()`.
- **Optional**: a hardcoded timezone offset (like this template's `+10/24`) is fragile — breaks
  across daylight saving changes, and is silently wrong for any viewer not in that timezone.
  Prefer a proper timezone conversion where the platform supports one.

## 8. Company reporting standards — visuals, colour, and performance

These are JBHIFI Group-wide report-authoring standards (from company guidance, not derived from
inspecting this template) — identical to the JB Hi-Fi and GRP docs, repeated here since they apply
regardless of which template a report descends from.

### Native visuals preferred over imported ones
- **Recommended**: prefer a native Power BI visual over an imported/custom one whenever an
  equivalent exists — e.g. the native **Button Slicer** instead of the imported **Chiclet Slicer**.
  Flag any imported visual and check whether a native equivalent would serve the same purpose
  before accepting it.
- **Note specific to this template**: several measure descriptions (`Delivered Sales Icon`,
  `Written Sales Icon`) explicitly say "use in Conditional formatting on Matrix vis" — see §10's
  Matrix guidance below; a Matrix-driven icon pattern here should still be weighed against the
  Table + Field Parameter alternative where the report's purpose allows it.

### Colour palette
- **Recommended**: prefer the JB colour theme shipped with the reporting template over ad-hoc
  colours. Where a non-brand colour is needed (status/traffic-light indicators, chart accents),
  use these standard hex codes rather than inventing new ones:

  | Purpose | Colour | Hex |
  |---|---|---|
  | Default text | Default Text Black | `#252423` |
  | — | Black | `#000000` |
  | Traffic light (text) | Soft Green | `#AAE6AA` |
  | Traffic light (text) | Soft Amber | `#F7DE6F` |
  | Traffic light (text) | Soft Red | `#FF8080` |
  | Traffic light (background) | Harsh Red | `#BF2020` |
  | Traffic light (background) | Harsh Amber | `#FFA500` |
  | Traffic light (background) | Harsh Green | `#3B803B` |
  | Chart | Orange | `#F2C80F` |
  | Chart | Light complimentary orange | `#FAE99F` |
  | Chart | Light Grey | `#BBBBBB` |
  | Chart | Dark Grey | `#2F2F2E` |
  | Chart | Gold | `#E8D166` |
  | Chart | Silver | `#B3B3B3` |

  Soft variants are for *text* on a light background; harsh variants are for status
  *backgrounds* (e.g. a KPI tile). Don't use a harsh-background hex as a text colour or vice versa.

  **This template's own icon/colour scheme, per its own measure descriptions**: `Delivered Sales
  Icon`/`Font Color` use "Gold ★ if above Budget, Black ● if above LY, Red ● if below both." Gold
  maps to the palette's `#E8D166`; "Black" and "Red" here should be checked against `#000000`/
  `#252423` and `#BF2020`/`#FF8080` respectively **once you can reach the upstream model** — per
  §0, the actual hex values are not visible from this file.

  **How to check compliance**: don't just eyeball a screenshot against the table above.
  1. Find any colour-switching measure and check what it actually computes for the current filter
     context — for this template, that means reaching the upstream `Sales Dataset` model (§0),
     not just reading the local `EXTERNALMEASURE` wrapper.
  2. Separately check what colour the visual *actually renders* — if it doesn't match what the
     measure computes, the visual likely isn't bound to that measure at all.
  3. Only after ruling that out, check whether the rendered colour matches the brand theme or the
     table above.

### Title, logo, and font-size hierarchy

- **Recommended**: report title and the JB/TGG logo should appear together in a consistent header
  band across every page of a report — a viewer shouldn't lose the report's identity when
  navigating between pages.
- **Recommended**: hold a consistent font-size hierarchy: report title largest, section/visual
  titles next, KPI card values sized for at-a-glance reading, body/table text smaller again, and
  slicer labels smallest.
- **Recommended**: one font family across the report.
- This is a report-canvas/PBIR concern — verifying it needs `powerbi-report-design`/
  `powerbi-report-authoring` against the actual pages, not this doc's MCP-only checks.

### Slicer and button layout

- **Recommended**: keep slicers grouped in one consistent panel/band per report. This template's
  own `EI Slicer Panel Context Text` measure (displayFolder `Context Text`) suggests an existing
  "Executive Insights" slicer panel pattern — follow that existing panel convention for new pages
  rather than introducing a differently-organized one.
- **Recommended**: this template's `Button Text` measures (`Location Selling Cat Button Default`,
  `Merch Cat Location Button Default`, etc.) drive dynamic button labels for drill/navigation
  buttons — a good pattern; keep new navigation buttons consistent with this dynamic-label
  approach rather than hardcoding button text.
- **Optional**: don't over-slice a single panel — consider a Field Parameter if a report
  accumulates many independent slicers.
- Confirm actual positioning via `powerbi-report-authoring`, not by inference from the model.

### Report/model performance and capacity

Fabric/Power BI Service capacity is shared across all reports on it — a poorly optimised report
degrades performance for everyone else on the same capacity, not just its own users.

- **Recommended**: measure before optimising — use **Performance Analyzer**. Standard tip: add a
  blank page, save and reopen the report (clears the visual + data-engine cache), start
  Performance Analyzer on the blank page, then navigate to the page under test.
- **Recommended**: avoid **Matrix** visuals where possible — several of this template's own
  measure descriptions explicitly mention "use in Conditional formatting on Matrix vis," so check
  whether those specific visuals could be Table + Field Parameter instead, per the report's actual
  needs.
- **Note specific to this template**: since almost all measures are `EXTERNALMEASURE`/DirectQuery
  (§0), the usual "flag overly complex local DAX" check mostly doesn't apply here — the compute
  cost and complexity live in the upstream `Sales Dataset` model, which is out of scope for a
  review of this file. If performance issues arise, they're at least as likely to be upstream
  model/DirectQuery-latency issues as local-report issues — don't assume the fix is always local.
- **Recommended**: check the model is aggregated to the lowest granularity the report actually
  needs.

Raise these as **recommended** findings unless a specific violation is severe enough to
independently justify **critical**.

## 9. Measure name must match its formula

**General standard (applies to any measure, in any report, regardless of template)**: a measure's
name is a promise about what it computes. Someone building a visual trusts the name, not the
expression behind it — if the two disagree, that's a defect independent of whether the DAX itself
is otherwise correct or error-free.

**This template's own confirmed example** (§7): `Report Refreshed Text` claims to show when data
was refreshed, but its formula only computes the current wall-clock time.

**Special case for this template (per §0)**: for any `EXTERNALMEASURE`-wrapped measure, the
formula itself isn't visible locally — you can only compare the name against the measure's `type`
and `description`, not its actual DAX. A name/description that's internally inconsistent (e.g. a
`... in Scope?` measure that isn't `Boolean`, or a `... Icon`/`... Color` measure that isn't a
`String`) is still checkable from this file; full formula-level mismatch detection needs the
upstream model.

Review rule:
- **Recommended**: for every measure reviewed, flag any name/formula (or, for externalized
  measures, name/type/description) mismatch found, including partial or ambiguous ones.
- **Critical** only when the mismatch is severe enough that trusting the name at face value would
  produce a wrong business or operational conclusion (as with `Report Refreshed Text`).

## Findings format

Report findings grouped by **critical / recommended / optional**, each citing the section above
and the proposed fix, and wait for user approval before applying any fix. For this template, §0
should always be raised first — it determines what the rest of the review can and can't actually
verify from this file alone.

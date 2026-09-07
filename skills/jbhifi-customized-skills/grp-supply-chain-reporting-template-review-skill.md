---
name: grp-supply-chain-reporting-template-review-skill
description: "Review standard for Power BI reports built from the GRP Supply Chain Reporting Template. Load this before reviewing, auditing, or approving a report derived from that template. Includes a confirmed-broken measure pattern specific to this template — check it before trusting any LY/LYTD comparative measure."
---

# GRP Supply Chain Reporting Template — Report Review Standard

This checklist is derived from a live inspection (via the `powerbi-modeling-stdio` MCP server,
connected to Power BI Desktop) of the **GRP Supply Chain Reporting Template** semantic model on
2026-09-04 (the same day
[jb-hifi-reporting-template-review-skill.md](./jb-hifi-reporting-template-review-skill.md) was
written from the JB Hi-Fi template).

**Shared lineage, confirmed**: the `Metrics` table (`lineageTag: 546afcca-...`) and the
`Sales Dollars` measure (`lineageTag: 9208a556-...`) carry identical `lineageTag`s to the JB Hi-Fi
template — same authoring history, same base `Sales` fact table shape (14 columns, byte-for-byte
matching column names). Sections 1–3, 8, and 9 of the JB Hi-Fi doc apply here largely unchanged;
this doc only calls out where GRP has **diverged** from that shared base, and does not repeat
what's identical. Read the JB Hi-Fi doc first.

## 0. Critical — confirmed broken LY/LYTD measures

**Verified directly from the model, not inferred**: `Sales Dollars LY` and `Sales Dollars LYTD`
both report:

```json
"state": "SemanticError",
"errorMessage": "USERELATIONSHIP function can only use the two columns references participating in relationship."
```

Root cause: both measures call
`USERELATIONSHIP('Sales'[Written Sales Date], 'Date'[Ly Day])`, but **`relationship_operations
List` returns zero relationships touching `Sales` at all** in this template copy — not even the
inactive one JB Hi-Fi has. `USERELATIONSHIP` requires the column pair to already participate in
some relationship (active or inactive); here it doesn't, so the DAX engine rejects the measure at
compile time. This almost certainly cascades to `Sales Quantity LY`, `Sales Quantity LYTD`, and
possibly the `YoY Var` measures that depend on the LY measures — check all of them, not just the
two confirmed above.

Review rule:
- **Critical, blocking**: before approving any report built from this template, run
  `measure_operations Get` on every `02. Comparative Measures` folder measure and check its
  `state` field. Any `SemanticError` measure must be fixed (most likely: add the
  `Sales[Written Sales Date]` → `Date[Ly Day]` inactive relationship, mirroring JB Hi-Fi's §5/§6)
  before the report ships — a report visual referencing a `SemanticError` measure will show a
  blank or an error tile, not silently fall back.
- Do not assume this is already fixed in a newer copy of the template — re-check `state` on these
  measures every time, since §1 below shows the template is still being actively edited
  (`Line of Business Colour` modified 2026-05-04, `Slicer Panel Background Colour` added
  2026-05-03) while this defect persists.

## 1. Sales fact table has no dimension relationships at all

JB Hi-Fi's `Sales` table has 9 active relationships to `Store`, `Product`, `Date`, `Employee`,
`Selling Channel`, `Promotions`, `Country`, and `Time Group`. **GRP's `Sales` table — same 14
columns, same data — has none.** `relationship_operations List` returns 10 relationships total in
this model, and every one of them is among `Employee`, `Line of Business`, `Product`, `Store`,
`Supplier`, `Product V2`, `Group Employee`, `Group Product`, `PBI User Security` — none involve
`Sales`.

This isn't just an oversight to flag gently — it's why §0's measures are broken, and it means
**no visual in a report built on this template will currently filter `Sales` by Store, Product,
Date, or anything else**, unless relationships are added first.

Likely cause, confirmed by column inspection: GRP's `Store` table uses a different key shape
(`Store Code`, `Store ID`, `Line Of Business Store ID`) than `Sales`' retained
`COUNTRY_STORE_ID` — the base `Sales` table was carried over unchanged from the single-brand JB
Hi-Fi shape, but `Store`/`Product` were rebuilt at group level with different keys, so even
auto-detect relationship discovery wouldn't find a match.

Review rule:
- **Critical, blocking**: do not approve a report from this template without first wiring `Sales`
  to at least `Date` (required for §0's fix) and to whichever dimensions the report actually
  slices by. This will likely require a bridge/mapping table translating
  `COUNTRY_STORE_ID` ↔ `Store Code`, `COUNTRY_STOCK_CODE` ↔ `STOCK_CODE`,
  `COUNTRY_EMPLOYEE_CODE` ↔ `EMPLOYEE_CODE`, `COUNTRY_ID` ↔ `LINE_OF_BUS_ID` — confirm with
  whoever owns the source data before inventing one.
- If you find a report derived from this template where `Sales` *is* wired up correctly, treat
  that as the reference and consider back-porting the relationships into the template itself.

## 2. `Date` table is a reduced as needed-column subset — not the full fiscal calendar

JB Hi-Fi's `Date` table has 109 columns (full Fiscal + Pnl calendars, Ly/Lly offsets for every
grain, `Is*` flags — see JB Hi-Fi doc §5). GRP's `Date` table has only **18 columns**:
`DATE_KEY`, `LY Day`, `LLY Day`, a handful of `Fiscal ...` grain columns
(`Day Of Fiscal Week`, `Fiscal Week End Date`, `Fiscal Month Of Fiscal Year`,
`Fiscal Week Of Fiscal Year`, `Fiscal Quarter Name`, `Fiscal Year Name`), a handful of `Pnl ...`
columns (`Pnl Year/Quarter Name`, `Pnl Year/Quarter Code`, `Pnl Calendar Month Name/Code`,
`Pnl Ly Day`, `Pnl Lly Day`, `Week Of Pnl Year`) — no `Is*` flags, no per-grain
start/end date columns beyond `Fiscal Week End Date`.

Review rule:
- **Critical**: before reusing a JB Hi-Fi-style DAX pattern that references a column like
  `Fiscal Year Start Date`, `Is Ytd`, or `Ly Week Start Date`, check it actually exists in *this*
  `Date` table — most of those columns are absent here and the measure will fail to compile.
- **Recommended**: if a report needs YTD/rolling-window logic beyond what `DATESYTD` (already used
  in `Sales Dollars YTD`, fiscal year-end `"6/30"`) and the surviving `Ly Day`/`Lly Day`/`Pnl Ly
  Day`/`Pnl Lly Day` columns support, either extend this `Date` table with the missing columns
  (matching JB Hi-Fi's naming) or confirm with the model owner whether the full calendar was
  deliberately trimmed for Supply Chain reporting.

## 3. `Line of Business` replaces `Country` as the segmentation dimension

GRP has no `Country` table at all. Instead, a `Line of Business` table
(`LINE_OF_BUS_ID`, `Line Of Business Code`, `Line Of Business Description`) is the hub: `Employee`,
`Product`, `Store`, `Supplier`, and `Product V2` all carry a `Many:One`, `OneDirection`
relationship into it.

`Line of Business Colour` (same measure, same `lineageTag` as JB Hi-Fi's `Line of Business
Colour`) has been edited to match — `SWITCH(MIN('Line of Business'[LINE_OF_BUS_ID]), 1, "#fff200",
2, "#fFf200")` — plus a **third branch stubbed out as a comment**:

```dax
SWITCH(MIN('Line of Business'[LINE_OF_BUS_ID]),
       1, "#fff200",   -- JBAU
       2, "#fFf200"   -- JBNZ
//       "#fFf200"       -- TGG
)
```

Review rule:
- **Recommended**: if a report needs to support the third line of business (TGG), note that the
  commented-out stub is **incomplete as written** — it's missing the `3,` case selector, so
  uncommenting the line as-is would not compile (`SWITCH` needs `value, result` pairs). Write the
  full `3, "<colour>",   -- TGG` pair rather than just uncommenting.
- **Recommended**: same check applies to `Line of Business Font Colour` if the report displays
  more than JBAU/JBNZ.
- **Critical**: `Report Name` in this template copy still reads the raw placeholder
  `"Report Name (MEASURE)"` (unmodified, same as JB Hi-Fi's shared default) — this specific open
  file is the template itself, not a customized report, so that's expected *here*; but any report
  saved from it must still customize this per JB Hi-Fi doc §1.

## 4. New metadata measure not present in JB Hi-Fi: `Slicer Panel Background Colour`

`00. Metadata` in GRP has one extra measure beyond JB Hi-Fi's set: `Slicer Panel Background
Colour`, a static hex string (`"#639470"`), added 2026-05-03 — i.e. after the shared base was
authored (April 2025). This is a newer template convention that hasn't (yet) propagated back to
the JB Hi-Fi template.

Review rule:
- **Optional**: if a report needs consistent slicer-panel theming and is JB Hi-Fi-based, consider
  whether this measure should be added there too — flag as a suggestion for the template owner,
  not something to silently port during an unrelated report review.

## 5. Group-level overlay dimensions: `Group Employee`, `Group Product`

Two tables extend the base `Employee`/`Product` dimensions with cross-brand attributes rather than
adding columns directly to the shared base tables:

- **`Group Employee`** — joined `One:One`, `BothDirections` to `Employee` (`EMPLOYEE_CODE` ↔
  `EMPLOYEE_ID`). Carries `Employee Full Name`, `Employee State Code`, `Employee Work Email`,
  `Employee Is Active`, plus `LINE_OF_BUSINESS_ID`/`LINE_OF_BUSINESS_EMPLOYEE_CODE`.
- **`Group Product`** — joined `Many:One`, `OneDirection` to `Supplier` only
  (`SUPPLIER_ID`). Carries `Product Brand`, `Product Model`, `Product Description`,
  `Supplier Name`/`Family`, product physical dimensions (`Product Length/Width/Height/Weight`,
  `Product Volume Cubic Metres`), and department/subgroup hierarchy
  (`Brand Product Department`, `Group Product Department`, etc.). It is **not** related to the
  base `Product` table directly.

Two of `Product`'s own relationships are **inactive**: `Product` → `Product V2` and `Product` →
`Supplier`. Only `Group Product` → `Supplier` is active. This suggests supplier/product-V2 lineage
is being routed through `Group Product` rather than `Product`'s direct links.

> A derived report's relationship list will differ from this template's — different fact tables
> mean different wiring. What's being checked is the *standard* (justified bidirectional paths,
> no ambiguous dual routes to the same dimension), not that the relationships match exactly.

Review rule:
- **Recommended**: follow this "Group X satellite table" pattern for any new cross-brand attribute
  — don't add group-level columns straight onto `Employee`/`Product`, which stay portable
  per-brand tables.
- **Recommended**: before reactivating `Product` → `Product V2` or `Product` → `Supplier`
  directly, check whether that creates an ambiguous second path alongside `Group Product` →
  `Supplier` — resolve deliberately (e.g. keep one path active) rather than leaving both active.

## 6. Additional dimensions not in JB Hi-Fi

`Supplier` (3 cols), `Returns App Statuses` (`Return Status Name/Sort Order/ID`, `Return Stage` —
4 cols), and `Product V2` (17 cols, parallel/successor schema to `Product`) exist only in this
template. Treat them as supply-chain-specific additions; there is no JB Hi-Fi equivalent to
diff against, so review them on general modeling-guidelines merit (star-schema fit, naming,
hidden-key convention per JB Hi-Fi doc §4) rather than against a shared baseline.

## 7. `PBI User Security` shape differs from JB Hi-Fi

JB Hi-Fi's RLS table has 8 columns keyed by store/area/region
(`COUNTRY_ID, STORE_ID, COUNTRY_STORE_ID, AREA_ID, REGIONAL_ID, AREA_MANAGER_ID, Store Email, Area
Manager Email`). GRP's has 5, keyed by employee and line of business instead
(`Line Of Bus ID, Employee ID, Employee Store, Employee Work Email, Employee State Code`) — a
different RLS grain (per-employee) suited to a Group/Supply-Chain audience rather than
per-store-area. Both are hidden (`isHidden: true`), which is the one invariant to check.

Review rule:
- **Critical**: this table must remain hidden, same as JB Hi-Fi doc §7. Don't assume its column
  shape should match JB Hi-Fi's — it's intentionally different here.
- Same DENY as JB Hi-Fi doc §7: don't touch RLS role membership as part of a review.

## 8. Company reporting standards — visuals, colour, and performance

These are JBHIFI Group-wide report-authoring standards (from company guidance, not derived from
inspecting this template) — identical to JB Hi-Fi doc §10, repeated here since they apply
regardless of which template a report descends from.

### Native visuals preferred over imported ones
- **Recommended**: prefer a native Power BI visual over an imported/custom one whenever an
  equivalent exists — e.g. the native **Button Slicer** instead of the imported **Chiclet Slicer**.
  Flag any imported visual and check whether a native equivalent would serve the same purpose
  before accepting it.

### Colour palette
- **Recommended**: prefer the JB colour theme shipped with the reporting template over ad-hoc
  colours (this is what the `Line of Business Colour` / `Line of Business Font Colour` /
  `Slicer Panel Background Colour` measures in §3/§4 exist to apply). Where a non-brand colour is
  needed (status/traffic-light indicators, chart accents), use these standard hex codes rather
  than inventing new ones:

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

### Report/model performance and capacity

Fabric/Power BI Service capacity is shared across all reports on it — a poorly optimised report
degrades performance for everyone else on the same capacity, not just its own users.

- **Recommended**: measure before optimising — use **Performance Analyzer**. Standard tip: add a
  blank page, save and reopen the report (this clears the visual + data-engine cache), start
  Performance Analyzer on the blank page, *then* navigate to the page under test — this captures
  true initial-load performance rather than a warm-cache number.
- **Recommended**: avoid **Matrix** visuals where possible — they're capacity-expensive. Prefer a
  **Table visual + Field Parameters** when the report needs to let the user change granularity —
  this template's own `Metrics (FP)` / `Dimensions (FP)` tables (JB Hi-Fi doc §8) are exactly this
  pattern; point report authors at that existing scaffolding rather than reaching for a Matrix.
- **Critical**: flag overly complex calculated columns or measures during review — they push
  compute cost into every report using the model. If a report's DAX is fighting the model instead
  of the model supporting the report, that's a sign to request a purpose-built Snowflake asset
  from the Reporting and Analytics team rather than compensating with heavier DAX. (This template
  already shows the cost of unresolved model gaps — see §0/§1.)
- **Recommended**: check the model is aggregated to the lowest granularity the report actually
  needs — e.g. don't import SKU-level rows if every visual reports at product-department level.
  This is a modeling-time decision (partition/M-query grain), not something to patch after the
  fact with `SUMMARIZE` in every measure.

Raise these as **recommended** findings unless a specific violation is severe enough to
independently justify **critical** (e.g. a Matrix visual driving a known capacity incident, or a
measure whose complexity is the confirmed root cause of a reported performance issue) — don't
default to critical just because a rule exists here.

## Findings format

Same as the JB Hi-Fi doc: report findings grouped by **critical / recommended / optional**, each
citing the section above and the proposed fix, and wait for approval before applying any fix. For
this template, §0 and §1 should always be raised first — they block any comparative measure and
any dimension-filtered visual from working correctly.

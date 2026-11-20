# JB NZ Services Insights — report context

Seed example for `skills/jbhifi-customized-skills/report-context-capture.md`.

## Report
- Purpose: attach rate, ADU and GP of add-on services (Telco, Apple Care, Extra Care, JB Extras,
  M365 and Internet Security, Installation) against the opportunities (primary product sales) they
  attach to, for JB Hi-Fi NZ. (Inferred from table and measure names; business owner unverified.)
- Pages (page.json, verified): Services Leaderboard, Services Breakdown, Services by Day,
  KPI Tooltip, Tooltip, Help and Guidance, Page 1.
- Services Leaderboard: Attach KPIs table, plus two overlapping matrices (Area > Store by Service
  Name; one visible, one hidden and swapped by bookmark).

## Source
- Open in Desktop as a PBIX under Downloads. A PBIP copy exists at `samples/JBNZ service/`
  (whether it matches the PBIX was not verified).
- Template lineage: not determined.
- Tables are Import mode, loaded from Fabric dataflows. Source IDs deliberately omitted.
  Note: an earlier copy of the model pointed at a different dataflow than the current one and
  served stale KPI rows (4 services instead of 7).

## Tables (verified from model)
- Facts: `Services Sales` (~3.12M rows; attach amounts, units, cost, Service, Service Type,
  Unique Reference), `Primary Sales` (~1.99M rows; one row per Unique Reference).
- Dimensions: `Store`, `Product`, `Date` / `Date Group`, `Country`, `Selling Channel`,
  `Sales Person`, `Service Type`, `KPIs`, `Sale Type` (Written / Delivered).
- `Metrics`: measure-only table (56 measures before our changes).
- Field parameters: `Store (FP)`, `Product (FP)`, `Time Period (FP)`.
- `Service Type` columns: Service Name, Service, Service Type, Sort Order. (A calculated
  `Attach Group` column was added 2026-11-12 and removed 2026-11-16, because Service Name now
  has M365 and Internet Security as separate values.)

## Relationships (non-obvious only; verified by relationship list)
- `Services Sales[Unique Reference]` <-> `Primary Sales[Unique Reference]`: many-to-many, both
  directions. `Primary Sales` has no direct link to `Store` or `Service Type`; it is reached
  through this path. `Primary Sales[Unique Reference]` is actually unique (row count equals
  distinct count).
- `Services Sales[Service]` -> `Service Type[Service]`: active, single direction.
- `Services Sales[Service Type]` <-> `Service Type[Service Type]`: inactive many-to-many, activated
  by `OppsXFilter` through `USERELATIONSHIP`.
- `KPIs[Attach Services Name]` <-> `Service Type[Service Name]`: many-to-many, both directions
  (verified 2026-11-16; briefly pointed at `Attach Group`, now removed).
- Written vs Delivered store and date keys exist as inactive relationships.

## Measure architecture (verified)
- Layers by display folder: 01. Base Metrics, 01. TY Metrics, 02. Comparative Metrics (PP),
  03. Variation Metrics (Var), KPIs, Periods, Labels, 00. Metadata.
- Pattern: Base measure -> TY measure wrapped as `IncludeBlankChannels(ApplySaleType([Base]))` ->
  PP via `ApplyPP` -> Var.
- DAX user-defined functions: `ApplySaleType`, `ApplyPP`, `IncludeBlankChannels`,
  `OppsTelcoFilter`, `OppsXFilter`, `VarBlank`.
- `Opp Sales Units/Amount` = `OppsTelcoFilter(OppsXFilter(SUM('Primary Sales'[...])))`.
  `OppsXFilter` (changed 2026-11-11): when the selected Service Name is Installation it uses the
  active Service relationship, so Installation splits into Visual / Computers & Comms; for every
  other Service Name it switches to the Service Type relationship (total per Service Type).
- Added 2026-11-12: `Attach GP Base`, `Attach GP`, `Attach Sales Amount Exc Gst (+Base)`,
  `Attach GP %` (GP = Exc-GST attach sales minus cost; % uses Exc-GST sales).

## Business rules
- Service Name -> Service: Telco {IFO, Plan Only, Sidekick}; Apple Care {Apple Care};
  Extra Care {EC Repair, EC Replace}; JB Extras {JB Extras};
  Installation {Computers & Comms, Visual, Others}. As of 2026-11-16 Service Name lists M365 and
  Internet Security as separate values (verified by DAX); the Service values under them were not
  re-queried (unverified). Earlier they were one Service Name, "M365 and Internet Security".
- Attach KPI targets (Mid / High) exist per Service Name in `KPIs` (7 services, verified
  2026-11-16); source keys are renamed in the KPIs model query (for example OEM is Internet
  Security, installation is Installation).
- Opportunity counts were identical for M365 and Internet Security on 2026-11-12 because they
  share one Service Type (not re-checked since the Service Name change).

## Known data quirks
- Computers & Comms rows had no attach units or amount in the earlier model copy (verified
  there; not re-checked in the current one).
- ~510 attach units sat under a blank Service Name group on 2026-11-12 (rows with no matching
  Service Type); not re-checked.
- Extra Care GP % showed 100% (cost blank?); Telco and JB Extras GP / GP % blank. Causes not
  investigated (unverified).

## Performance notes (verified 2026-11-12, no date filter, one run each)
- Area x Store x Service Name with `[Attach]` + `[ADU]`: ~36 s. About 22 s is `[Opp Sales Units]`,
  ~31 s for a plain `SUM('Primary Sales'[Primary Sale Units])` at that grain; the attach side is
  cheap.
- Likely cause: the many-to-many path above (unverified as the cause). Proposed fix, not tested:
  make `Primary Sales[Unique Reference]` the "one" side, keep bidirectional.

## Open issues
- Attach KPIs table: whether it now shows M365 and Internet Security as separate rows has not
  been checked in the report (visuals aren't readable from a PBIX through MCP).
- Relationship change above awaiting approval and testing.

## Changes made
- 2026-11-10 to 2026-11-12: see `docs/claude-activity-log.md` entries for GP measures,
  OppsXFilter, KPI Attach Group changes and the reapply on the new file; 2026-11-16 entry for
  removing `Attach Group` and returning the KPI relationship and measures to Service Name.

## Last verified
- 2026-11-12, via the live model through MCP (relationship list, DAX queries) and PBIR page files.
  Not re-checked on 2026-11-16 when this file was written. Re-verify if the PBIX was rebuilt or
  replaced.

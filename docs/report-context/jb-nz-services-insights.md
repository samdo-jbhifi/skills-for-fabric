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
- `Service Type` columns: Service Name, Service, Service Type, Sort Order, plus calculated
  `Attach Group` (added 2026-11-12).

## Relationships (non-obvious only; verified by relationship list)
- `Services Sales[Unique Reference]` <-> `Primary Sales[Unique Reference]`: many-to-many, both
  directions. `Primary Sales` has no direct link to `Store` or `Service Type`; it is reached
  through this path. `Primary Sales[Unique Reference]` is actually unique (row count equals
  distinct count).
- `Services Sales[Service]` -> `Service Type[Service]`: active, single direction.
- `Services Sales[Service Type]` <-> `Service Type[Service Type]`: inactive many-to-many, activated
  by `OppsXFilter` through `USERELATIONSHIP`.
- `KPIs[Attach Services Name]` <-> `Service Type[Attach Group]`: many-to-many, both directions
  (was `Service Name` until 2026-11-12).
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
  M365 and Internet Security {M365, Internet Security}; Installation {Computers & Comms, Visual,
  Others}.
- Attach KPI targets (Mid / High) exist per `Attach Group` in `KPIs`; the source keys were
  renamed in the model query (for example OEM is Internet Security).
- Opportunity counts are identical for M365 and Internet Security because they share one
  Service Type (verified in the model; behaviour before our changes was not confirmed).

## Known data quirks
- Computers & Comms rows had no attach units or amount in the earlier model copy (verified
  there; not re-checked in the current one).
- ~510 attach units sit under a blank `Attach Group` (rows with no matching Service Type).
- Extra Care GP % showed 100% (cost blank?); Telco and JB Extras GP / GP % blank. Causes not
  investigated (unverified).

## Performance notes (verified 2026-11-12, no date filter, one run each)
- Area x Store x Service Name with `[Attach]` + `[ADU]`: ~36 s. About 22 s is `[Opp Sales Units]`,
  ~31 s for a plain `SUM('Primary Sales'[Primary Sale Units])` at that grain; the attach side is
  cheap.
- Likely cause: the many-to-many path above (unverified as the cause). Proposed fix, not tested:
  make `Primary Sales[Unique Reference]` the "one" side, keep bidirectional.

## Open issues
- Attach KPIs table visual still uses `Service Name` rows; swap to `Attach Group` pending.
- Relationship change above awaiting approval and testing.

## Changes made
- 2026-11-10 to 2026-11-12: see `docs/claude-activity-log.md` entries for GP measures,
  OppsXFilter, KPI Attach Group changes and the reapply on the new file.

## Last verified
- 2026-11-12, via the live model through MCP (relationship list, DAX queries) and PBIR page files.
  Not re-checked on 2026-11-16 when this file was written. Re-verify if the PBIX was rebuilt or
  replaced.

# Claude Activity Log

Append-only. See `CLAUDE.md` § Activity Logging for the required entry format and rules.

> **Note on the entries below**: these are backfilled retroactively on 2026-09-14, covering
> significant actions from earlier in this session. Exact clock times for past actions were not
> recorded as they happened, so entries use the date confirmed via session date-change notices
> (2026-09-04, 2026-09-08, 2026-09-09, 2026-09-14) with time marked as not recorded, rather than
> inventing a precise timestamp. All entries from this point forward will have accurate times.

---

## [2026-09-04, time not recorded]

### Request
Confirm whether MCP is connected to Power BI, and investigate the `powerbi-modeling-stdio` server configuration.

### Rationale
User asked to verify Power BI MCP connectivity before doing any model work.

### Actions Performed
- Read `.mcp.json` to inspect configured MCP servers.
- Ran `powerbi-modeling-mcp.exe --help` and a manual start to diagnose a "Connection closed" error.
- Identified the fix: server needed `--readwrite`/`--skip-confirmation` flags (applied later, see 2026-09-09 entry).

### Tools Used
- Read (`.mcp.json`)
- Bash/PowerShell (running the exe directly, `--help`)

### Files Read
- `.mcp.json`

### Files Modified
- None (at this point)

### Files Created
- None

### MCP Calls
- None (server not yet reachable)

### Result
Confirmed the `powerbi-modeling-stdio` binary existed and started correctly standalone; root caused the harness-side "Connection closed" as unrelated to the binary itself.

### Next Planned Action
Connect to live Power BI Desktop templates and begin semantic-model review work.

### Notes
None.

---

## [2026-09-04, time not recorded]

### Request
Inspect the JB Hi-Fi and GRP Supply Chain reporting templates and create JBHIFI-specific review-standard skill docs.

### Rationale
User wanted reusable review checklists derived from live inspection of the actual templates, not generic best practice.

### Actions Performed
- Connected to `JB Hi-Fi Reporting Template` and `GRP Supply Chain Reporting Template` via `powerbi-modeling-stdio`.
- Inspected tables, relationships, measures, columns, RLS, and model settings on both.
- Authored `skills/jbhifi-customized-skills/jb-hifi-reporting-template-review-skill.md`, `grp-...md`, and the router `SKILL.md`.
- Registered the new skill in `CLAUDE.md` and `AGENTS.md`.

### Tools Used
- `powerbi-modeling-stdio` MCP (connection_operations, table_operations, relationship_operations, measure_operations, column_operations, model_operations)
- Write / Edit (new skill docs)
- Read (existing skill doc conventions for formatting consistency)

### Files Read
- `skills/semantic-model-authoring/SKILL.md` (for formatting convention)

### Files Modified
- `CLAUDE.md`, `AGENTS.md` (added JBHIFI report review skill entries)

### Files Created
- `skills/jbhifi-customized-skills/SKILL.md`
- `skills/jbhifi-customized-skills/jb-hifi-reporting-template-review-skill.md`
- `skills/jbhifi-customized-skills/grp-supply-chain-reporting-template-review-skill.md` (later renamed)

### MCP Calls
- Tool: `powerbi-modeling-stdio` (multiple operations)
- Purpose: full model inventory of both templates (tables, relationships, measures, columns, RLS, model settings)
- Scope: two full semantic models (explicitly requested/required — building the review standard needed complete inventories)
- Estimated response size: Large (cumulative across the session)
- Estimated token impact: High (cumulative)
- Context availability: not previously available
- Reusable in current session: Yes (formed the basis of all later reviews)

### Result
Two review-standard docs and a router skill created, capturing real defects (e.g. GRP's confirmed broken `SemanticError` measures) and conventions from the live templates.

### Next Planned Action
Continue refining the review docs based on user feedback (generalization, company guidance sections).

### Notes
This was a large, explicitly-requested exploratory task (building new skill docs from scratch) — the Large/High estimate reflects that, not routine review work.

---

## [2026-09-08, time not recorded]

### Request
Delete the local `JBHIFI_GROUP_TEMPLATES/` folder (640MB, confirmed cloud backup exists), then recreate it empty.

### Rationale
User confirmed a cloud copy existed and wanted local disk space back.

### Actions Performed
- Confirmed no external backup uncertainty remained (user explicitly confirmed cloud copy).
- Deleted `JBHIFI_GROUP_TEMPLATES/` (recursive, 136 files, 640.3 MB).
- Recreated it as an empty folder per follow-up request.

### Tools Used
- PowerShell (`Remove-Item -Recurse -Force`, `New-Item -ItemType Directory`)

### Files Read
- None

### Files Modified
- None

### Files Created
- None (folder recreated empty)

### MCP Calls
- None

### Result
Folder removed and recreated empty as requested; local disk space freed.

### Next Planned Action
None at the time.

### Notes
Destructive operation — explicitly confirmed with the user before proceeding (cloud backup confirmed).

---

## [2026-09-08 to 2026-09-09, time not recorded]

### Request
Review `JB AU Extra Care Error Tracker - Support Office` against the JB Hi-Fi skill doc; refine the skill docs for over-specificity (generalize "Sales" fact-table references, add company colour/visual/performance guidance, add "measure name must match formula" section); investigate and document the TGG PBI Style Template as a third template family.

### Rationale
User wanted a real review of a live report, then wanted the review skill docs themselves audited and improved based on findings from that review, then wanted a third, structurally distinct template documented.

### Actions Performed
- Connected to `JB AU Extra Care Error Tracker - Support Office` (.pbix); found 12 broken measures (`SemanticError`/`DependencyError`), wrong `Report Name`, naming convention violations, and (via a found stale PBIP export) a colour-binding regression on the report banner.
- Cross-referenced findings against all 38 visuals in an on-disk PBIR export to confirm which measures were genuinely unused.
- Edited all three `jbhifi-customized-skills` docs: generalized fact-table/relationship language, added star-schema baseline notes, added company guidance (native visuals, colour palette, performance) to both per-template docs, added the "measure name must match its formula" section to both, extended the router's "no unused" principle to tables/assets, added an "ask lineage first" section and broadened GRP detection to a `Group`/`GRP` naming pattern.
- Renamed `grp-supply-chain-reporting-template-review-skill.md` → `grp-reporting-template-review-skill.md` per user request, and updated all cross-references and the router table (with a caveat that the real `.pbip` file name wasn't independently renamed).
- Connected to `TGG PBI Style Template v1.6` (required installing Node.js via winget for the report-authoring skill, later set aside per user instruction not to use it yet); inspected its `EXTERNALMEASURE`/DirectQuery-federation architecture; authored `tgg-reporting-template-review-skill.md`; updated the router to add TGG as a third lineage and its `EXTERNALMEASURE` detection signal.
- Applied the first round of approved fixes (C1 delete 12 broken measures, C2 fix `Report Name`, C3 rebind the banner shape's colour) to the `.pbix` Desktop session (port 65010).

### Tools Used
- `powerbi-modeling-stdio` MCP (full range of operations)
- Grep / Read (PBIR JSON inspection across a found stale export)
- Edit (skill doc revisions)
- PowerShell (winget Node.js install, later not used further per user instruction)

### Files Read
- `skills/jbhifi-customized-skills/*.md` (all three, multiple passes)
- Various `.Report/definition/**/*.json` files under a found PBIP export in `Downloads\Marketplace Analysis\`

### Files Modified
- `skills/jbhifi-customized-skills/SKILL.md`
- `skills/jbhifi-customized-skills/jb-hifi-reporting-template-review-skill.md`
- `skills/jbhifi-customized-skills/grp-reporting-template-review-skill.md` (renamed + edited)

### Files Created
- `skills/jbhifi-customized-skills/tgg-reporting-template-review-skill.md`

### MCP Calls
- Tool: `powerbi-modeling-stdio` (table/measure/column/relationship/model operations, plus write operations for C1–C3)
- Purpose: full review of `JB AU Extra Care Error Tracker`, full review of `TGG PBI Style Template`, applying approved fixes
- Scope: full model inventories (explicitly needed to build/verify the review) plus targeted writes for the approved fixes
- Estimated response size: Large (cumulative)
- Estimated token impact: High (cumulative)
- Context availability: partially available (some findings reused from earlier reviews)
- Reusable in current session: Yes

### Result
Three review-standard docs, all generalized per user feedback; TGG documented as a third template family; first round of fixes applied to the (since-closed) `.pbix` session.

### Next Planned Action
Re-verify fixes against the project's PBIP copy.

### Notes
Node.js was installed via winget (system-level change, done with explicit user approval) to support the report-authoring skill; that skill's use was then paused per user instruction ("do not use authoring skill... node").

---

## [2026-09-14, time not recorded]

### Request
Enable Power BI MCP write mode without confirmation prompts; re-review `JB AU Extra Care Error Tracker` (this time via `samples/...pbip`, with both MCP and PBIR access); investigate and resolve the "Duplicate of Sales and Return Errors" question; re-apply and extend fixes (C1/C2/C3, then C5/R3, with R5/C4 corrected after re-verification).

### Rationale
Prior fixes turned out not to have been saved into the project (applied to a since-closed `.pbix` session); user wanted them correctly re-applied to the actual project file, plus the newly-identified duplicate page investigated and its content brought in line with the skill standard.

### Actions Performed
- Edited `.mcp.json` to add `--readwrite --skip-confirmation` to the `powerbi-modeling-stdio` server args (with explicit user approval, after diagnosing that write confirmations were being silently declined).
- Reconnected to `JB AU Extra Care Error Tracker - Support Office` (this time the `samples/...pbip` project); confirmed it was the pre-fix copy.
- Re-applied C1 (deleted 12 confirmed-unused broken measures, re-verified unused against the current PBIR first), C2 (recreated `Report Name` with corrected text, preserving `lineageTag`), and C3 (rebound the banner shape's `visualContainerObjects.background.color` from a hardcoded `ThemeDataColor` back to the `[Line of Business Colour]` measure) — sequenced carefully around the user's Desktop save to avoid Desktop overwriting the PBIR edit.
- Investigated "Duplicate of Sales and Return Errors": confirmed it's a report page (not a model table), currently the report's active/default page, containing a superset of the original page's content; reported findings and asked the user how to proceed.
- Per user instruction ("leave the page name as is, fix content per JB skills"): re-verified `Return Errors`/`Return Errors_1`/`Return Errors_2` usage (found only `_1` unused — corrected an earlier assumption that all three were dead) and deleted only `Return Errors_1`; renamed 4 columns (`Employee_ID`→`Employee ID`, `Primary_SKUs`→`Primary SKUs`, and the two `ERROR_...` ALL_CAPS columns to Title Case); added `displayFolder: "03. Error Measures"` to 8 domain measures via delete+recreate (the MCP server's `Update` operation is broken in this build); corrected an earlier finding that the `htmlContent` custom visual was "unused" — it's the sanctioned Kada widget integration, confirmed via its query binding.
- Verified all dependent calculated columns/measures remained in a `Ready` state after each rename.

### Tools Used
- Edit (`.mcp.json`)
- `powerbi-modeling-stdio` MCP (connection, table, column, measure, relationship, model operations)
- Grep / Read (PBIR pages, bookmarks, `pages.json`, visual.json files)
- PowerShell (verifying TMDL contents on disk after the user's Desktop save)

### Files Read
- `.mcp.json`
- `samples/JB AU Extra Care Error Tracker - Support Office.SemanticModel/definition/tables/Metrics.tmdl`
- `samples/JB AU Extra Care Error Tracker - Support Office.Report/definition/pages/pages.json`
- `samples/.../pages/cb3d0945e904d7000b29/page.json`
- `samples/.../pages/ReportSectionb6d3226b94089875b0d9/visuals/2721f1cbe68e512e3c06/visual.json`
- Multiple other `.Report` page/visual JSON files (targeted greps)

### Files Modified
- `.mcp.json` (added `--readwrite --skip-confirmation`)
- `samples/JB AU Extra Care Error Tracker - Support Office.Report/definition/pages/ReportSection5fb92df44622200b6c12/visuals/e0c23b387e6aac20d6e5/visual.json` (C3 colour rebind)

### Files Created
- None (this entry)

### MCP Calls
- Tool: `powerbi-modeling-stdio` (table_operations, measure_operations, column_operations, connection_operations, relationship_operations, model_operations)
- Purpose: re-verify pre-fix state, apply C1/C2/C5/R3 fixes, verify post-fix state
- Scope: targeted, named-object operations throughout (specific tables/measures/columns by name) — no full-model or full-table-list scans beyond the initial and final verification passes
- Estimated response size: Medium (many small named-object calls, not a handful of large ones)
- Estimated token impact: Medium
- Context availability: partially available (model shape known from earlier reviews; live state re-verified before each write)
- Reusable in current session: Yes

### Result
C1, C2, C3 confirmed re-applied and persisted (pending the user's Desktop save, confirmed via TMDL read); C5 and R3 applied and verified; R5 and C4 corrected after re-verification rather than acted on incorrectly. `Duplicate of Sales and Return Errors` page identified and left un-renamed per user instruction; its content not yet reconciled with the model-level fixes (open item).

### Next Planned Action
Check the duplicate page's own banner shape for the same colour-binding issue as the original page; address R1 (Matrix visual) if approved; await user decision on R2/R4/R6 (business-context items).

### Notes
Per-call before/after cost reporting (as `CLAUDE.md` specifies) was not done consistently during the fix-application sequence in this session — batched summaries were given instead. Flagged to the user; no change made without a subsequent explicit decision.

---

## [2026-09-14, time not recorded]

### Request
Create and backfill `docs/claude-activity-log.md`.

### Rationale
`CLAUDE.md`'s Activity Logging section requires this file be maintained; it had not yet been created. User confirmed to create and backfill now.

### Actions Performed
- Created `docs/` directory and `docs/claude-activity-log.md`.
- Backfilled entries for the significant actions identifiable from this session, using known session dates (exact historical clock times not recorded).

### Tools Used
- Write (`docs/claude-activity-log.md`)

### Files Read
- None

### Files Modified
- None

### Files Created
- `docs/claude-activity-log.md`

### MCP Calls
- None

### Result
Activity log created with backfilled entries covering this session's major actions to date.

### Next Planned Action
Log future significant actions going forward, per-entry, as they occur.

### Notes
Backfilled entries are grouped by task/milestone rather than by individual tool call, given the volume of calls across this session — this keeps the log readable while still capturing every significant action category the rules require (MCP calls, file changes, model changes). Exact historical timestamps are approximated to the known session date; this is noted at the top of the file rather than presented as precise.

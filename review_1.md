##cost $8

Review: samples\JB AU Extra Care Error Tracker - Support Office.pbip
Routed via jbhifi-customized-skills/SKILL.md → JB Hi-Fi lineage (has Country, Selling Channel, Date Group, Metrics, Kada Widget, FP tables; no EXTERNALMEASURE/DirectQuery, no Group/GRP tables) → applied jb-hifi-reporting-template-review-skill.md. This time I had both sides — MCP for the model, and the live .Report/ PBIR files (since it's a PBIP), so the visual checks are real, not deferred.

Two things to flag before the findings
1. This is the pre-fix copy. None of the 9 Sept fixes are in it — 26 measures (not 14), Report Name still "JB NZ…", columns still underscore/ALL_CAPS. Those fixes were applied to the .pbix Desktop session (port 65010, since closed) and never saved into this project. So everything we fixed is back.

2. I got something wrong last review, and this copy proves it. I called the htmlContent… custom visual "orphaned/unused." That was a methodology error on my part — my regex "visualType": "[a-zA-Z]+" excluded digits, and the type name is htmlContent443BE3AD55E043BF878BED274D3A6855. It was never orphaned; it's placed on the Guidance page. Correcting that below.

Critical
C1 — 12 broken measures (§3/§5/§11). Sales Dollars / Sales Quantity = SemanticError (reference Sales Amount Inc Gst / Quantity, which don't exist — the table has Primary…/Extra Care… variants). Sales Dollars LY = SemanticError on USERELATIONSHIP. The YoY/YTD set cascades as DependencyError.

C2 — Report Name is wrong and now unused (§1 + router's no-unused rule). Still "JB NZ Extra Care Error Tracker" on the AU report. And grepping all pages for "Property": "Report Name" returns nothing — in the July export it was at least bound to a container title on the Guidance page; in this copy it's referenced by no visual at all.

C3 — Purple banner: root cause proven (§10 Colour). The banner shape e0c23b387e6aac20d6e5 ("Shape: Slicer Header"):

July export: visualContainerObjects.background.color → Measure: [Line of Business Colour] (computes #fef200, JB yellow).
This copy: → ThemeDataColor { ColorId: 5, Percent: 0.4 }. Theme dataColors[5] = #744EC2 — stock Power BI purple, lightened 40%. That's the purple in your screenshot.
So someone unbound the measure and replaced it with a stock theme colour. #744EC2 is neither the brand yellow nor anything in the documented JB palette. The model's mechanism is intact — the visual just stopped using it. This is exactly the §10 "how to check compliance" case: the real defect isn't "wrong colour", it's "the report stopped using the mechanism the model provides."

C4 — Imported custom visual in use (§10 Native visuals). htmlContent443BE3AD55E043BF878BED274D3A6855 is placed on the Guidance page (2721f1cbe68e512e3c06). Per §10, check whether a native textbox/shape covers the need before accepting an imported visual.

C5 — Naming convention (§4). Employee_ID, Primary_SKUs visible + underscore_case; ERROR_INCORRECT_EXTRA_CARE_STOCK_CODE / ERROR_EXCLUDED_PRIMARY_STOCK_CODE ALL_CAPS alongside four Title-Case siblings doing the same job.

Recommended
R1 (§10 Performance) — a pivotTable (matrix) sits on the main page (a68a1915657913ad1729). §10 says avoid Matrix where possible; the model already ships Metrics (FP)/Dimensions (FP) for the Table + Field Parameters alternative.
R2 (§11) — Excluded Product sums only ERROR_EXCLUDED_PRIMARY_STOCK_CODE (Primary line) in a model that also has an Extra Care line; the name doesn't disambiguate scope.
R3 (§2) — 8 domain measures still have no displayFolder.
R4 (§1) — Line of Business Colour still returns #fef200 on all three SWITCH branches.
R5 — Return Errors / _1 / _2: three near-identical calculated columns still present.
R6 (§6) — PBI User Security → Store is BothDirections; confirm it's intentional.
Cleared
Sales and Return Errors wired to 8 dimensions (§6 ✓) · PBI User Security isHidden: true (§7 ✓) · culture: en-AU, __PBI_TimeIntelligenceEnabled: "0" (§9 ✓) · Date table retains all 109 columns (§5 ✓).


created new content in claude.md global with summary for:


content: "Claude Code Global Instructions", covering:

Cost Tracking Rules — before any MCP tool call: explain why it's needed, why context is insufficient, what's expected, and a size estimate (Small/Medium/Large) — with confirmation required if Large. After every call: report tool name, purpose, objects accessed/returned, size/token-impact estimates, context availability, and reusability.
Cost Reduction Rules — avoid broad reads/scans, reuse context and prior MCP results, prefer named/targeted lookups, don't spawn subagents or do optional exploration unnecessarily, warn before token-heavy ops, confirm before High-impact ones.
Power BI and MCP Cost Controls — never pull whole models/tables/columns/measures/relationships/TMDL/pages unless explicitly requested or technically required; prefer targeted single-object queries; explain before any broad model operation; log every MCP call.
Activity Logging — maintain an append-only docs/claude-activity-log.md (create docs/ if missing), with a specific entry format (Request/Rationale/Actions/Tools/Files Read/Modified/Created/MCP Calls/Result/Next Planned Action/Notes), logged for every significant action — MCP calls, scans, multi-file reads, file create/modify/delete, running scripts/tests/builds/deployments, config changes, Power BI/DAX/SQL/PQ changes, or Medium/High-impact ops. Skipped for trivial no-tool responses.
User-Facing Summary — after every significant action: what was done, tools/files involved, estimated token impact, whether the log was updated.
Approval Rules — confirm before High-impact ops, full-model/all-object reads, full repo scans, multiple subagents, bulk changes, deletions, destructive changes, or deploy/publish. No confirmation needed for targeted named-object reads or appending to the log.
General Working Rules — smallest sufficient operation, no unmodified files unless required, safest low-cost interpretation when ambiguous, narrower follow-up queries if a tool over-returns.
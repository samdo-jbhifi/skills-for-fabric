# Report Context Capture

Record a short, reusable summary of a Power BI report/model the first time it is analysed through
`powerbi-modeling-stdio` MCP, so later sessions can start from the summary instead of
re-inspecting the model.

This is a procedure, not a review standard. It does not judge the report against a template
(see the per-template docs for that) and it does not replace `semantic-model-authoring`.

## When it applies

- You are about to analyse, or have just analysed, a report/model through MCP modelling and no
  summary for it exists yet in `docs/report-context/`.
- The user asks to "record context", "summarise this report", or "save what we know about this
  report" for any report.

It does not apply to throw-away questions that only touch one measure.

## Step 1 — Look for an existing summary first

1. Look for `docs/report-context/<report-name>.md` (kebab-case of the report name, for example
   `jb-nz-services-insights.md`). Use a targeted Glob, not a repository scan.
2. If one exists: read it and use it as the starting point. Only re-verify what the current task
   depends on. Check its "Last verified" date and "Model state" line; if the model has plausibly
   changed since, say so and re-verify the affected sections.
3. If none exists: carry on with the task, collecting the facts below as you go. Do not run extra
   discovery purely to fill the template.

## Step 2 — What to capture

Concise, about one page. Use exactly these headings so summaries stay comparable.

| Section | Content |
|---|---|
| **Report** | Name, business purpose, intended audience/owner (if known), pages worth knowing about |
| **Source** | File type (PBIP or PBIX) and location, template lineage if known (JB Hi-Fi, GRP, TGG, unknown) |
| **Tables** | Fact vs dimension role of each important table, one line each. Mention field parameters, calculation tables and measure tables |
| **Relationships** | Only the non-obvious ones: many-to-many, bidirectional, inactive (and which measure activates them), and any that bridge fact tables |
| **Measure architecture** | How measures are layered (for example Base, TY, PP, Var), shared DAX user-defined functions and what each does, display-folder convention |
| **Business rules** | Mappings and definitions the DAX depends on (for example which Service rolls up to which Service Name). State the rule, not the full DAX |
| **Known data quirks** | Blank or null columns, unmatched keys, counts that surprised you |
| **Performance notes** | Measured timings and the cause if known |
| **Open issues** | Anything found but not resolved |
| **Changes made** | One line per change with a pointer to the matching `docs/claude-activity-log.md` entry. Do not duplicate the log |
| **Last verified** | Date and how: "MCP live model" or "PBIP files" |

Keep it to what a future session would otherwise have to rediscover. Leave out anything the repo
or git history already records, and leave out full measure lists and column inventories.

## Step 3 — Capture rules (all mandatory)

1. **Verified or unverified.** Every statement is either checked from source (add the means in a
   few words, for example "DAX query" or "relationship list") or explicitly marked
   `(unverified)`. Never record an assumption as fact.
2. **Nothing sensitive.** Do not store credentials, tokens, secrets, connection strings, email
   addresses, or workspace, dataflow, dataset or tenant IDs. If a source reference is needed,
   write the item's display name instead. If you meet such data while inspecting, flag that it
   exists and do not repeat it.
3. **Nothing session-specific.** Do not store Desktop ports, process IDs or MCP connection names;
   they change with every Desktop restart.
4. **Do not store row-level data.** Aggregate counts and shapes only, unless a specific value is
   itself the finding.
5. **One report per file.** Never merge two reports into one summary.

## Step 4 — Offer, then save only on confirmation

1. After the first analysis, show the draft summary in the reply and ask: "Save this to
   `docs/report-context/<name>.md`?"
2. Write the file only after the user says yes. CLAUDE.md Review Mode forbids creating files
   during reviews unless explicitly requested, so a review must never create this file silently.
3. If the user edits the draft, save their version.
4. After writing, append an entry to `docs/claude-activity-log.md` (file created) in the standard
   format. Updating an existing summary is also a file modification and is logged the same way.

## Step 5 — Keeping a summary honest

- When a later session finds a statement is wrong, correct it in place, change `Last verified`,
  and log the modification.
- When the report is rebuilt or replaced (a new PBIP, a recreated file), treat the old summary as
  a draft: re-verify tables, relationships and measure architecture before relying on it.
- Do not let a summary stand in for inspection when the task is a review, audit or fix. Use it to
  focus the inspection, not to skip it.

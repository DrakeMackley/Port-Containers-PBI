# Agentic Power BI — demo runsheet

Fill in `<PLACEHOLDERS>` before running.

**Sequencing rule:** model changes appear in Desktop live; report file edits do not.
Segments 1–6 run with Desktop open. Save and close after segment 6. Segments 8–9 edit files
on disk. Reopen Desktop at segment 10 for the reveal.

---

## Pre-flight

- [ ] `git log --oneline` shows a baseline commit
- [ ] `Test-Path ".\<PROJECT>.Report\definition"` returns `True`
- [ ] `powerbi-modeling-mcp` visible in Copilot Chat tools
- [ ] Model set to GPT-5 or Claude Sonnet 4.5
- [ ] PBIP open in Power BI Desktop, agent connected
- [ ] Manual title banner built on one page (see segment 8)

---

## 1 — Model analysis *(read-only, ~3 min)*

> Describe this semantic model. List the tables, all measures with their DAX, relationships with
> cardinality and filter direction, and any date/time columns. Tell me whether a date table is
> marked as a date table. Read-only — change nothing.

**Watch for:** invented tables or columns. If anything is wrong here, stop and fix context.

---

## 2 — Improvement recommendations *(read-only, ~4 min)*

> Based on what you just read, give me your top five improvements to this model, ranked by impact.
> For each: the specific object, what's wrong, what you'd change. Be concrete about this model —
> skip anything that would apply to any Power BI model. Don't implement anything yet.

**Watch for:** generic best-practice boilerplate. Grounded suggestions name real objects.

---

## 3 — First measure *(~5 min)*

> Create one measure: year-over-year percentage change on `<BASE MEASURE>`. Use DIVIDE, return
> BLANK when there's no prior-period value. Add a description in business language. Then execute
> it with a DAX query grouped by year and show me the returned values. Don't call it done until
> I've seen numbers.

Write-approval prompt fires here. This is the canary — date table problems surface now, on one
measure instead of six.

---

## 4 — Remaining measures *(~7 min)*

> Now create the rest: `<LIST>`. Same pattern — DIVIDE for ratios, blank handling, business-language
> description on each. In the DAX itself, include developer comments: a `//` header line stating what
> the measure answers, and inline comments on any non-obvious filter context or variable. Validate
> each with a query and give me one table of all results. Stop and ask if any measure needs a column
> that doesn't already exist.

---

## 5 — Comment the pre-existing measures *(~8 min, highest risk)*

Three prompts. Do not collapse them.

**5a — capture baseline**

> Before changing anything: execute every measure in this model and give me a table of measure name
> and current value at total grain. Save this as our baseline for comparison.

**5b — apply comments**

> Now add developer comments to the DAX of every pre-existing measure in the model. For each: a `//`
> header line stating what business question it answers, and inline comments on any non-obvious
> filter context, variable, or CALCULATE modifier. Do not change any evaluation logic — comments and
> whitespace only. List every measure you modified.

**5c — regression check**

> Re-run all measures and compare against the baseline table. Show both columns side by side and
> flag any value that changed. Any difference means the logic was altered and must be reverted.

**If anything moved:** `git checkout -- <PROJECT>.SemanticModel` after saving, or restore from your
folder copy. This is why the baseline commit exists.

---

## 6 — Dependency audit *(read-only, ~5 min — bridge between halves)*

> Read the PBIR files under `<PROJECT>.Report\definition\pages\` and build a table showing which
> measures and columns are used by which visuals on which pages. Include the visual type. Then tell
> me which measures are not referenced by any visual, and which pages depend on the most measures.

Pure file reading. Nothing here can break. Note aloud that this was impossible with PBIX — it is
the clearest argument for the format change.

---

## 7 — Save and close Desktop

Save in Desktop (flushes model changes to TMDL), commit, then close Desktop before editing report files.

```powershell
git add . ; git commit -m "Model: new measures + comments"
```

---

## 8 — Title banner *(~8 min)*

Prerequisite: build one banner manually in Desktop before the demo — a white rectangle plus a
textbox — so the agent has a known-good schema to copy rather than inventing one.

> On page `<PAGE FOLDER>` there's a rectangle and a textbox forming a title banner. Replicate that
> exact pattern onto every other page, preserving position, size and formatting. On each page, set
> the textbox content to that page's own `displayName` from its `page.json`. List the files you'll
> create or modify before you start.

**Risk:** new `visual.json` files with schema errors are blocking errors — Desktop won't open the
report at all. Recovery is `git checkout -- <PROJECT>.Report`.

---

## 9 — Font *(~4 min)*

> In the theme file at `StaticResources\RegisteredResources\<THEME>.json`, change the font family
> across all textClasses to `<FONT>`. Leave sizes and colours alone. List which textClasses changed.

**If nothing changes on reopen:** check for page-level overrides in `page.json`, which win over the theme.

---

## 10 — Reopen Desktop — the reveal

Both report changes land at once. If Desktop reports a blocking error, it names the offending file.

---

## 11 — Model documentation *(~6 min, closer)*

> Generate a Markdown document giving complete professional documentation for this semantic model.
> Include a mermaid diagram of the table relationships; every measure with its DAX and a
> business-language explanation of the logic; any row-level security filters; and the data sources,
> inferred by analysing the Power Query. Write it to `docs/model-documentation.md`.

Writes a new file, touches nothing existing. Ends the demo with a durable artifact rather than a
screenshot.

---

## 12 — Diff review *(~5 min)*

```powershell
git status
git diff --stat
git diff -- "*.tmdl"
git diff -- "*.json"
```

The TMDL diff shows what the agent wrote into your model; the JSON diff shows the report edits.
This is where you find out whether the output is idiomatic or merely valid.

---

## Recovery

| Problem | Fix |
|---|---|
| Desktop won't open the report | `git checkout -- <PROJECT>.Report`, reopen |
| Measure values changed after commenting | `git checkout -- <PROJECT>.SemanticModel`, re-save from Desktop |
| Theme edit did nothing | Check page-level overrides; check the resource is registered |
| Model change missing from diff | Not saved in Desktop yet |
| Everything is broken | `git reset --hard HEAD` |

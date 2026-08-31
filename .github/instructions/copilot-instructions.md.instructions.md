---
description: Describe when these instructions should be loaded by the agent based on task context
# Project context: agentic Power BI development

This repository is a Power BI Project (PBIP) saved in **PBIR** format. You will be asked to
modify the semantic model and the report layer. Read these rules before any edit.

Replace every `<PLACEHOLDER>` below before first use.

---

## Layout

```
<PROJECT_NAME>\
├── <PROJECT_NAME>.pbip
├── <PROJECT_NAME>.Report\
│   ├── definition.pbir                 # pointer only: datasetReference + version
│   ├── definition\                     # PBIR report definition
│   │   ├── report.json                 # report-level formatting, themeCollection  [EDITABLE]
│   │   ├── version.json
│   │   ├── pages\
│   │   │   ├── pages.json
│   │   │   └── [pageName]\
│   │   │       ├── page.json           # page formatting, incl. canvas background  [EDITABLE]
│   │   │       └── visuals\[visualName]\visual.json
│   │   ├── bookmarks\
│   │   └── reportExtensions.json
│   └── StaticResources\
│       └── RegisteredResources\
│           └── <THEME_NAME>.json       # custom theme                              [EDITABLE]
└── <PROJECT_NAME>.SemanticModel\
    └── definition\                     # TMDL
```

Active theme file: `<PROJECT_NAME>.Report\StaticResources\RegisteredResources\<THEME_NAME>.json`

---

## Do not edit

- `<PROJECT_NAME>.Report\report.json` (root level, if present) — PBIR-Legacy, external editing unsupported.
  The editable one is `definition\report.json`. These are different files with the same name.
- `mobileState.json`
- `semanticModelDiagramLayout.json`
- `.pbi\localSettings.json`
- Anything under `StaticResources\SharedResources\BaseThemes\` — Desktop-generated, regenerated on save.

Do not add new files to `RegisteredResources\`. New resources require an entry in `report.json`,
which does not support external editing during preview. Only edit resources already registered
by Power BI Desktop.

---

## Semantic model work

Use the `powerbi-modeling-mcp` tools. Do not hand-edit TMDL unless asked.

1. **Read before writing.** Inspect existing tables, columns, measures and relationships and state
   what you found. Do not assume a field exists because its name is conventional.
2. **Prefer measures over calculated columns.** Measures are metadata-only and apply instantly.
   Calculated columns force a model recalculation. Create a calculated column only when explicitly
   asked, and only on a small dimension table.
3. **Validate every measure you create** by executing it with `dax_query_operations` and reporting
   the actual returned values. A measure is not done until it has returned a sane result.
   Syntactically valid but semantically wrong DAX is the failure mode to guard against.
4. **Use `DIVIDE`** rather than `/` for any ratio. Handle blanks explicitly.
5. **Write a description** for every measure you create, in business language, explaining what it
   answers rather than restating the DAX.
6. Report each change as: object name, what it does, the DAX, and the validated result.

---

## Report and theme work

1. **Fonts are theme-level.** Edit `textClasses` in the theme file. One file, applies report-wide.
2. **Canvas background is page-level.** It lives in each `page.json`. Changing it across the report
   means editing every page file.
3. **Page-level settings override the theme.** If a page has an explicit background value, a
   theme-level change will not appear. Check the page files before concluding a theme edit failed.
4. **Respect the JSON schema.** Every PBIR file declares its `$schema` at the top. A wrong property
   name or type produces a blocking error that prevents Power BI Desktop from opening the report.
5. Preserve existing formatting and key order in files you edit. Change only what was asked.

---

## Workflow rules

- **The file on disk is the source of truth for the report layer.** If Power BI Desktop has unsaved
  changes, they are invisible to you. Ask the user to save before you read report files.
- **Model writes via Desktop connection do not reach TMDL until the user saves in Desktop.** Say so
  when it matters.
- Never batch model changes and report changes in a single turn. Complete one, confirm, then move on.
- Before any multi-file edit, list the files you intend to touch and wait for confirmation.
- After each change, summarise the diff in one or two lines. Do not restate the full file.

---

## Known traps

| Symptom | Cause |
|---|---|
| Theme edit has no visible effect | Page-level override, or resource not registered in `report.json` |
| Desktop won't open the report | Schema violation in an edited PBIR file — check `$schema` validation in VS Code |
| Agent's model change isn't in the git diff | Connected to Desktop, not saved yet |
| Page folder names are opaque GUIDs | Use Desktop's *Copy object name* (Report settings → Report objects) to map a page to its folder |
| New measure returns blank | Field reference wrong, or filter context assumption incorrect — re-validate with a query |
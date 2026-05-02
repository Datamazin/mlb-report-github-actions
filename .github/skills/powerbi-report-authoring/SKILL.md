---
name: powerbi-report-authoring
description: Guide to develop Power BI Reports in PBIR format. Use this skill for any development operation against a Power BI Report PBIR file format including (1) Creating new reports on top of semantic models, (2) Editing visuals, pages, and bookmarks, (3) Aligning and laying out visuals, (4) Rebinding reports to different semantic models, (5) Deploying reports to Fabric workspaces, (6) Exporting reports from workspaces. Do NOT use for semantic model development or TMDL (use powerbi-semantic-model), or workspace/pipeline administration (use fabric-cli).
---

# Power BI Report (PBIR) Skill

This skill provides guidance on how to develop Power BI reports using the PBIR (Power BI Report) JSON file format.

## Critical

- Schema‑aware: validate JSON against the declared `$schema`; call out violations and propose fixes.
- Understand the [PBIP file structure](../powerbi-semantic-model/references/pbip.md).

## Relationship to Other Skills

- **powerbi-semantic-model**: Use for semantic model development (tables, measures, DAX). This skill handles report layout, visuals, and pages only.
- **fabric-cli**: Use for workspace operations, deploying, and exporting report definitions.

## Pre-development: Understand the Report

Before making any changes to an existing report, always gather context first:

1. **Verify PBIR format** — Confirm the report has a `definition/` folder with `report.json`, `pages/`, and `version.json`.
2. **List all pages** — Read `pages/pages.json` to understand the page structure and order.
3. **List visuals per page** — For each page, list the visual folders and read each `visual.json` to understand visual types, positions, and field mappings.
4. **Identify the semantic model** — Read `definition.pbir` to understand which semantic model the report is connected to (`byPath` or `byConnection`).
5. **Check theme and resources** — Review `StaticResources/RegisteredResources/` for custom themes and images.

## PBIR file format

Example of a report folder using `PBIR` format:

```text
Report/
├── StaticResources/ # stores report resources such as images, custom themes,... 
│   ├── RegisteredResources/
│   │   ├── logo.jpg
│   │   ├── CustomTheme4437032645752863.json
├── definition/ # stores the entire report definition: pages, visuals, bookmarks
│   ├── bookmarks/
│   │   ├── Bookmark7c19b7211ada7de10c30.bookmark.json
│   │   ├── bookmarks.json
│   ├── pages/
│   │   ├── 61481e08c8c340011ce0/
│   │   │   ├── visuals/
│   │   │   │   ├── 3852e5607b224b8ebd1a/
│   │   │   │   │   ├── visual.json
│   │   │   │   │   ├── mobile.json
│   │   │   │   ├── 7df3763f63115a096029/
│   │   │   │   │   ├── visual.json
│   │   │   ├── page.json
│   │   ├── pages.json
│   ├── version.json
│   ├── report.json
├── semanticModelDiagramLayout.json # Copy of the semantic model diagram layout the report is connected to.
└── definition.pbir # Overall definition of the report and core settings. Also holds the reference to the semantic model of the report, it's possible to rebind the report to a different semantic model by updating this file.
```

## Name property

All report objects have a `name` property. For example see the `visual.json`:

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/visualContainer/2.1.0/schema.json",
  "name": "5868707ac858bcbe007a",
  "position": {
    "x": 32.118081180811807,
    "y": 216.32472324723247,
    "z": 4000,
    "height": 492.16236162361622,
    "width": 366.52398523985238,
    "tabOrder": 2000
  },
  "visual": {
    "visualType": "barChart",
    "drillFilterOtherVisuals": true
  }
}
```

By default PBIR folder uses the internal `name` property of pages, visuals and bookmarks in the files or folders. For example consider the following path of a visual:

  `Sales.Report\definition\pages\89a9619c7025093ade1c\visuals\5acb1caf298449a8acb4\visual.json`

  `89a9619c7025093ade1c` is the name of the page and `5acb1caf298449a8acb4` the name of the visual.

## Task: Create new report on top of semantic model

1. **Understand the semantic model** — Before creating any report, ensure you have a good understanding of the semantic model: its tables, key measures, dimensions, and date/calendar table.
2. **Create or reuse PBIP folder** — Create a new PBIP report folder structure or reuse an existing one. Follow the [PBIP structure](../powerbi-semantic-model/references/pbip.md).
3. **Copy template files** — Copy `assets/templateReport/report/definition` and `assets/templateReport/report/StaticResources` to the `*.Report` folder.
4. **Configure the semantic model reference** — Update `definition.pbir` to point to the target semantic model (use `byPath` for local, `byConnection` for workspace).
5. **Adapt visuals to the semantic model** — Load [template-report-kb.md](assets/templateReport/template-report-kb.md) and follow its instructions to map each visual to the semantic model fields (title, topCard, dateSlicer, barChart, timeSeries).
6. **Validate** — Run the [Post-development validation](#post-development-validate-changes) steps.

## Task: Edit an existing report

Use this workflow when the user wants to add/modify/remove visuals, pages, or bookmarks in an existing report.

1. **Locate the report** — Determine source (PBIP folder or export from workspace) and access the PBIR files.
2. **Run Pre-development discovery** — Follow the [Pre-development](#pre-development-understand-the-report) steps to understand the current report structure.
3. **Plan changes** — Based on the user request, identify which files need to be created, edited, or removed.
4. **Execute changes** — Apply modifications:
   - If adding visuals: create a new folder under the page's `visuals/` directory with a `visual.json` file. Use the correct `$schema` and set appropriate `position`, `visual.visualType`, and field mappings.
   - If editing visuals: modify the `visual.json` — update field mappings, position, or visual type.
   - If adding pages: create a new page folder under `pages/`, add a `page.json`, and update `pages/pages.json` with the new page entry.
   - If removing objects: delete the folder/file and update parent index files (e.g., `pages.json`, `bookmarks.json`).
5. **Validate** — Run the [Post-development validation](#post-development-validate-changes) steps.

## Task: Align Power BI Report visuals

1. **Verify PBIR format** — Confirm there is a `definition/` folder with the report pages.
2. **For each page**, perform the following:
   1. **Inspect all visual.json files** — Read the `position` property of each visual (`x`, `y`, `height`, `width`, `z`, `tabOrder`) and take into consideration the page dimensions (default 1280x720) in `page.json` to understand the current layout and identify misalignments or overlaps.
   2. **Build a wireframe** — Map out each visual's X/Y position and dimensions to understand the current layout.
   3. **Infer the layout grid** — Identify rows and columns in the current arrangement. Group visuals into rows and columns based on similar `y` (for rows) and `x` (for columns) values.    
   4. **Apply consistent alignment:**
      - Ensure horizontal and vertical distribution is even across rows and columns.
      - If a row has only one visual, expand its width to fill the row.
      - When possible, make all visuals in the same row have the same height and width.
   5. **Update position values** — Edit each `visual.json` with the corrected `position` properties.

## Task: Rename folders to improve readability

1. **Identify target objects** — The internal `name` property is used by default in folder and file names. You can rename folders for: pages, visuals, and bookmarks.
2. **Rename the folder** — Change the folder name to a human-readable name (e.g., `mainPage`, `salesBarChart`, `dateFilter`).
3. **Verify references** — Ensure no other files reference the old folder name (internal `name` property in JSON files remains unchanged — only the folder name changes).

## Task: Export a report from Workspace

1. **Create folder structure** — Set up the PBIP folder structure following the [PBIP guidelines](../powerbi-semantic-model/references/pbip.md).
2. **Export using Fabric CLI** — Use the `fabric-cli` skill to export the report code definition from the workspace.
3. **Place files correctly** — Ensure the exported definition goes into the `[Name].Report/definition/` local folder.
4. **Verify** — Confirm the exported folder contains `report.json`, `version.json`, and `pages/` with at least one page.

## Task: Import/Deploy a report to a Workspace

1. **Update `definition.pbir`** — **CRITICAL:** When deploying a report to a workspace, the `definition.pbir` must use a `byConnection` configuration targeting a workspace semantic model:

    ```json
    {  
      "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
      "version": "4.0",
      "datasetReference": {
        "byConnection": {      
          "connectionString": "semanticmodelid=[SemanticModelId]"
        }
      }
    }
    ```

2. **Deploy using Fabric CLI** — Use the `fabric-cli` skill to deploy. You can deploy with a new name if needed.
3. **Verify deployment** — Confirm the report appears in the target workspace and opens correctly.

## Task: Rebind Power BI Report to a different semantic model

### Target is a local PBIP folder

1. **Edit `definition.pbir`** — Configure a `byPath` property using a relative reference to the semantic model folder:

    ```json
    {
      "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
      "version": "4.0",
      "datasetReference": {
        "byPath": {
          "path": "../Sales.SemanticModel"
        }
      }
    }
    ```

2. **Verify field references** — After rebinding, check that visual field mappings (`Entity` and `Property` references in `visual.json` files) are still valid against the new semantic model.

### Target is in a Fabric workspace

1. **Find the semantic model ID** — Use Fabric CLI or workspace API to locate the target semantic model ID.
2. **Edit `definition.pbir`** — Configure a `byConnection` property targeting the semantic model ID:

    ```json
    {  
      "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
      "version": "4.0",
      "datasetReference": {
        "byConnection": {      
          "connectionString": "semanticmodelid=[SemanticModelId]"
        }
      }
    }
    ```

3. **Verify field references** — Ensure visual field mappings are compatible with the new semantic model.

## Post-development: Validate Changes

After any report modification, always verify your work:

1. **Validate file structure** - Ensure all required files and folders are present and correctly named. (See [PBIR file format](#pbir-file-format)).
1. **Validate JSON schemas** — Ensure all edited JSON files are valid against their declared `$schema` URL.
2. **Check visual field mappings** — Verify that `Entity` (table name) and `Property` (column/measure name) references in `visual.json` files match the connected semantic model.
3. **Verify page index** — Confirm `pages/pages.json` lists all page folders and the order is correct.
4. **Check visual positions** — Ensure no overlapping visuals and that positions make sense within the page dimensions.
5. **Validate `definition.pbir`** — Confirm the semantic model reference (`byPath` or `byConnection`) is correctly configured.
6. **Run BPA custom rules** — Run the [custom rules analysis](#task-analyze-report-against-best-practices) using `scripts/pbi-inspector-custom-rules.json` and address any **Error**-level findings before finalizing.

## Task: Analyze report against best practices

**IMPORTANT:** Always use the `scripts/bpa.ps1` PowerShell script to run PBIInspector (FabInspector) for report validation. Do NOT manually implement rule checks — the tool handles all validation logic defined in the JSON rules files.

Run the script `scripts/bpa.ps1` against the report definition. Two rule sets are available:

- **`scripts/bpa-rules-report.json`** — Generic PBIR best practice rules (default)
- **`scripts/pbi-inspector-custom-rules.json`** — Custom report rules including accessibility checks (non-default titles, alt-text, visual headers), UX rules (slicer search empty, first tab default), performance rules (TopN/Advanced filters, object counts), and design standards (theme colors, page scroll)

If no specific BPA rules are mentioned, run **both** rule sets. If a specific rule set is requested, use that one.

1. **Run BPA** — Execute the Best Practice Analysis script with each rule set:

    ```powershell
    # Run generic rules
    scripts/bpa.ps1 -reports [path to the report definition folder] -rulesFilePath scripts/bpa-rules-report.json

    # Run custom report rules (accessibility, UX, performance, design standards)
    scripts/bpa.ps1 -reports [path to the report definition folder] -rulesFilePath scripts/pbi-inspector-custom-rules.json
    ```

2. **Report findings** — Categorize results by severity levels (Critical, High, Medium, Info) and propose fixes.

### Custom rules reference (pbi-inspector-custom-rules.json)

| Rule ID | Severity | What it checks |
|---------|----------|----------------|
| `REMOVE_CUSTOM_VISUALS_NOT_USED` | Error | Unused custom visuals registered in the report |
| `REDUCE_VISIBLE_VISUALS_ON_PAGE` | Warning | More than 20 visible visuals per page |
| `REDUCE_OBJECTS_WITHIN_VISUALS` | Warning | More than 12 data fields assigned to a single visual |
| `REDUCE_TOPN_FILTERING_VISUALS` | Error | More than 4 visuals using TopN filtering per page |
| `REDUCE_ADVANCED_FILTERING_VISUALS` | Warning | More than 4 visuals using Advanced filtering per page |
| `REDUCE_NUMBER_OF_PAGES` | Warning | More than 10 pages in the report |
| `AVOID_SHOW_ITEMS_NO_DATA` | Warning | Visuals with "Show items with no data" enabled |
| `TOOLTIP_DRILLTHROUGH_HIDDEN` | Error | Tooltip/Drillthrough pages that are not hidden |
| `ENSURE_THEME_COLOURS` | Warning | Visuals using hardcoded hex colors instead of theme colors |
| `ENSURE_NO_VERTICAL_SCROLL` | Warning | Pages taller than 720px causing vertical scroll |
| `ENSURE_NON_DEFAULT_TITLES` | Error | Visuals without a meaningful custom title |
| `ENSURE_ALTTEXT` | Error | Non-decorative visuals missing alt-text for screen readers |
| `FIRST_TAB_DEFAULT` | Error | Active page is not the first page in the report |
| `NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_1` | Error | Visual headers enabled on slicers/navigators |
| `NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_2` | Error | Visual headers enabled on textboxes/shapes/buttons |
| `SLICER_SEARCH_INPUT_SHOULD_BE_EMPTY` | Error | Slicers with search text pre-filled |
| `NO_VISUAL_LEVEL_FILTERS_VISIBLE_IN_FILTER_PANEL` | Error | Visual level filters visible in the filter pane when filter panel is enabled (only page/report level filters should be visible) |

## Error Handling

- **Invalid JSON schema**: Read the `$schema` URL, validate the file structure against it, and fix any missing or extra properties.
- **Broken field references**: If a visual references a table or column that doesn't exist in the semantic model, update the `Entity` and `Property` values to match valid semantic model objects.
- **Deployment failure**: Verify that `definition.pbir` uses `byConnection` (not `byPath`) when deploying to a workspace. Check that the semantic model ID is correct.
- **Missing pages or visuals after export**: Verify the export completed fully. Re-export if the `pages/` folder is missing subfolders.
- **Template adaptation issues**: If the template report visuals don't render correctly, verify that field mappings use exact table and measure names from the semantic model (case-sensitive).

## References

**External references** (request markdown when possible):

- [PBIR docs](https://learn.microsoft.com/en-us/power-bi/developer/projects/projects-report?tabs=v2%2Cdesktop#pbir-format)

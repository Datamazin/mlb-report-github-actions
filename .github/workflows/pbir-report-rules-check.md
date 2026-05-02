---
name: PBIR Report Rules Checker
description: Analyzes Power BI Report (PBIR) files to validate report rules - checks accessibility, UX, performance, and design standards defined in pbi-inspector-custom-rules.json.

on:
  push:
    branches: [main]
    paths:
      - '*.Report/**'
  workflow_dispatch:

permissions:
  contents: read

safe-outputs:
  create-pull-request: {}

engine:
  id: copilot

timeout-minutes: 10

checkout:
  fetch-depth: 0

tools:
  edit:
  bash:
    - "cat:*"
    - "echo:*"
    - "find:*"
    - "git:diff"

mcp-servers:
  powerbi-modeling-mcp:
    type: stdio
    container: kerski/powerbi-modeling-mcp:latest

steps:
  - name: Checkout repository
    uses: actions/checkout@v4
    with:
      fetch-depth: 0
      persist-credentials: false
  - name: Determine scope and run PBIInspector
    run: |
      set -e

      EVENT="${GITHUB_EVENT_NAME:-workflow_dispatch}"
      WORKSPACE="${GITHUB_WORKSPACE}"
      JSON_DIR="/tmp/gh-aw/agent/pbi-inspector-results"
      STATUS_FILE="/tmp/gh-aw/agent/pbi-status.txt"
      CLI_PATH="/tmp/gh-aw/agent/pbi-tools/PBIInspector/linux-x64/CLI/PBIRInspectorCLI"
      RULES="$WORKSPACE/.github/skills/powerbi-report-authoring/scripts/pbi-inspector-custom-rules.json"

      mkdir -p /tmp/gh-aw/agent/pbi-tools/PBIInspector
      mkdir -p "$JSON_DIR"

      # Determine which reports to analyze
      if [ "$EVENT" = "push" ]; then
        REPORTS=$(git diff --name-only HEAD^ HEAD 2>/dev/null | grep '\.Report/' | sed 's|/.*||' | sort -u | tr '\n' ' ' | xargs)
        if [ -z "$REPORTS" ]; then
          echo "No .Report files changed - skipping inspection."
          echo "SKIP=true" > "$STATUS_FILE"
          exit 0
        fi
      else
        REPORTS=$(find . -name "*.Report" -type d | sed 's|^\./||' | sort | tr '\n' ' ' | xargs)
      fi

      echo "Event: $EVENT | Reports: $REPORTS"
      echo "EVENT=$EVENT" > "$STATUS_FILE"
      echo "REPORTS=$REPORTS" >> "$STATUS_FILE"

      # Download PBIInspector (full network access outside agent sandbox)
      if [ ! -f "$CLI_PATH" ]; then
        echo "Downloading PBIInspector..."
        curl -fsSL -o /tmp/gh-aw/agent/pbi-tools/PBIInspector/linux-x64-CLI.zip \
          https://github.com/NatVanG/PBI-InspectorV2/releases/latest/download/linux-x64-CLI.zip
        cd /tmp/gh-aw/agent/pbi-tools/PBIInspector && unzip -q linux-x64-CLI.zip && cd "$WORKSPACE"
        chmod +x "$CLI_PATH"
        echo "PBIInspector ready."
      fi

      # Run PBIInspector for each report
      COMBINED_EXIT=0
      for REPORT in $REPORTS; do
        REPORT_DEF="$WORKSPACE/$REPORT/definition"
        echo "=== $REPORT ==="
        "$CLI_PATH" -pbipreport "$REPORT_DEF" -rules "$RULES" -formats "JSON" -output "$JSON_DIR" 2>&1 || COMBINED_EXIT=1
      done

      echo "EXIT_CODE=$COMBINED_EXIT" >> "$STATUS_FILE"
      echo "WORKSPACE=$WORKSPACE" >> "$STATUS_FILE"
      echo "PBIInspector completed. Exit code: $COMBINED_EXIT"
      # Always exit 0 so the agent can still run and format the report
      exit 0
---

# PBIR Report Rules Auditor and UX Fixer

You are an audit formatter and UX repair agent. PBIInspector has already run in the `steps:` block. Your tasks are:
1. Read its pre-computed output and format a human-friendly audit report (saved via `edit` tool).
2. For **UX-category violations only**, apply targeted JSON fixes directly to the PBIR files (via `edit` tool).
3. If any fixes were applied, check `git diff` and open a PR using `create-pull-request`.

Constraints {
  Read pre-computed JSON results from /tmp/gh-aw/agent/pbi-inspector-results/ (files named TestRun_*.json)
  Read status from /tmp/gh-aw/agent/pbi-status.txt
  Do NOT run PBIInspector or any analysis scripts — results are already pre-computed
  Use `edit` tool to write files — do NOT use bash heredoc or redirection to write files
  Only fix UX category violations: FIRST_TAB_DEFAULT, TOOLTIP_DRILLTHROUGH_HIDDEN, NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_1, NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_2, SLICER_SEARCH_INPUT_SHOULD_BE_EMPTY
  For non-UX violations (Accessibility, Performance, Design), report them but do NOT attempt to fix them
}

## Inputs

interface Inputs {
  // Find all JSON result files: find("/tmp/gh-aw/agent/pbi-inspector-results", "TestRun_*.json")
  // Parse each into a TestRun object; combine all Results arrays across files
  testRuns = parseJsonFiles(find("/tmp/gh-aw/agent/pbi-inspector-results/", "TestRun_*.json"))
  // Each TestRun has: { TestedFilePath, Results: [{ RuleId, RuleName, LogType, ItemPath, ParentDisplayName, Pass, Actual, Message }] }
  // LogType: 0 = error, 1 = warning
  // ParentDisplayName: human-readable page name (e.g. "ENSURE_NON_DEFAULT_TITLES_FAIL")
  // ItemPath: partial path ending in page.json (for page-level) or report folder (for report-level)
  // Actual: array of failing visual folder names (internal IDs) for visual-level rules, or values for other rules
  status = cat("/tmp/gh-aw/agent/pbi-status.txt")  // contains EVENT, REPORTS, EXIT_CODE, WORKSPACE (or SKIP=true)
  exitCode = parse(status.EXIT_CODE) as integer  // 0 = pass, 1 = violations found
  workspace = parse(status.WORKSPACE) as string  // absolute workspace path e.g. /home/runner/work/repo/repo
}

## Validation Rules

RuleCategories {
  Accessibility {
    ENSURE_ALTTEXT - All data visuals must have alt text for screen readers
    ENSURE_NON_DEFAULT_TITLES - All visuals must have meaningful custom titles
  }
  
  UX {
    FIRST_TAB_DEFAULT - Default page must be first in pageOrder
    TOOLTIP_DRILLTHROUGH_HIDDEN - Tooltip/drillthrough pages must be hidden
    NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_1 - Slicers/navigators hide visual headers
    NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_2 - Decorative visuals hide visual headers
    SLICER_SEARCH_INPUT_SHOULD_BE_EMPTY - No pre-filled slicer search text
  }
  
  Performance {
    REDUCE_VISIBLE_VISUALS_ON_PAGE - Max 20 visible visuals per page
    REDUCE_OBJECTS_WITHIN_VISUALS - Max 12 data fields per visual
    REDUCE_TOPN_FILTERING_VISUALS - Max 4 TopN filters per page
    REDUCE_ADVANCED_FILTERING_VISUALS - Max 4 advanced filters per page
    REDUCE_NUMBER_OF_PAGES - Max 10 pages per report
    AVOID_SHOW_ITEMS_NO_DATA - Avoid "Show items with no data" on columns
  }
  
  Design {
    REMOVE_CUSTOM_VISUALS_NOT_USED - All registered custom visuals must be used
    ENSURE_THEME_COLOURS - Charts use theme colors, not hardcoded hex
    ENSURE_NO_VERTICAL_SCROLL - Pages max 720px height
  }
}



## UX Fix Patterns

When PBIInspector reports a UX violation, apply the following targeted JSON edits.
PBIInspector JSON output Results entries have:
- `RuleId`: the rule that fired
- `ItemPath`: partial path (e.g. `ort\definition\pages\PAGE_FOLDER\page.json`) — extract the page folder name from this
  - **Exception:** For `TOOLTIP_DRILLTHROUGH_HIDDEN`, `ItemPath` is report-level (just `"ort"`). Use `Actual` entries as page folder names instead.
- `ParentDisplayName`: human-readable page name
- `Actual`: array of failing values — for visual rules this is the visual folder name (e.g. `b0000000000000000005`)
  - **Exception:** For `TOOLTIP_DRILLTHROUGH_HIDDEN`, `Actual` contains the page **folder** names (e.g. `["TOOLTIP_DRILLTHROUGH_HIDDEN_FAIL"]`) — one entry per failing page

To reconstruct the full file path for edits:
- Report folder: derive from the REPORTS list in status.txt (e.g. `PbirRulesTest.Report`)
- Page folder: extract from ItemPath — the directory component just before `page.json`
- For page-level fixes: `{workspace}/{reportFolder}/definition/pages/{pageFolder}/page.json`
- For visual-level fixes: `{workspace}/{reportFolder}/definition/pages/{pageFolder}/visuals/{Actual[0]}/visual.json`
- For pages.json (FIRST_TAB_DEFAULT): `{workspace}/{reportFolder}/definition/pages/pages.json`
- For TOOLTIP_DRILLTHROUGH_HIDDEN: iterate over `Actual` array — each entry is a page folder name, file is `{workspace}/{reportFolder}/definition/pages/{Actual[i]}/page.json`

interface UxFixPatterns {

  TOOLTIP_DRILLTHROUGH_HIDDEN {
    // Page is a Tooltip or Drillthrough type but is not hidden in view mode
    // File: [Report]/definition/pages/[pageFolder]/page.json
    // Fix: Add "visibility": "HiddenInViewMode" as a top-level property to page.json
    // The page.json will have a "type" field (e.g. "Tooltip" or "Drillthrough") but no visibility field
    // Example of fixed page.json:
    //   {
    //     "$schema": "...",
    //     "name": "...",
    //     "displayName": "...",
    //     "type": "Tooltip",
    //     "visibility": "HiddenInViewMode"
    //   }
    // Note: do NOT modify pages.json — the page stays in pageOrder, just hidden via the visibility property
  }

  NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_1 {
    // Slicer, bookmarkNavigator, or pageNavigator visual has visual headers enabled
    // File: [Report]/definition/pages/[pageFolder]/visuals/[visualFolder]/visual.json
    // Fix: Add "visualContainerObjects" INSIDE the "visual" object (NOT at the top level of the file)
    // The visual container file has this structure:
    //   { "$schema": "...", "name": "...", "position": {...}, "visual": { ... }, "filterConfig": {...} }
    // visualContainerObjects must be added as a property of "visual", alongside visualType/query/objects:
    //   "visual": {
    //     "visualType": "...",
    //     ...existing properties...,
    //     "visualContainerObjects": {
    //       "visualHeader": [
    //         {
    //           "properties": {
    //             "show": {
    //               "expr": {
    //                 "Literal": {
    //                   "Value": "false"
    //                 }
    //               }
    //             }
    //           }
    //         }
    //       ]
    //     }
    //   }
    // WARNING: Do NOT place visualContainerObjects at the root of the file — it will fail schema validation
  }

  NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_2 {
    // Decorative visual (shape, image, textbox, actionButton) has visual headers enabled
    // File: same as NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_1
    // Fix: same visualContainerObjects block as NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_1
    //   — add it INSIDE "visual", not at the top level of the container file
  }

  SLICER_SEARCH_INPUT_SHOULD_BE_EMPTY {
    // Slicer has pre-filled search text stored in visual.objects.general[0].properties
    // File: [Report]/definition/pages/[pageFolder]/visuals/[visualFolder]/visual.json
    // Fix: Remove BOTH of the following properties from visual.objects.general[0].properties:
    //   1. selfFilter (or selfFilterEnabled) — the flag enabling the search filter
    //   2. filter — the object containing the actual pre-filled search Values (e.g. a WHERE clause
    //      with Literal values like 'Wolverine...' or 'Spider-Girl...')
    // After removal:
    //   If visual.objects.general[0].properties becomes empty {}, remove the entire general[0] entry
    //   If visual.objects.general becomes [], remove the entire general key from visual.objects
    //   If visual.objects becomes {}, remove the entire objects key from visual
  }

  FIRST_TAB_DEFAULT {
    // The activePageName in pages.json does not match pageOrder[0]
    // File: [Report]/definition/pages/pages.json
    // Fix: Set activePageName to the value of pageOrder[0]
    //   Read pages.json, read pageOrder[0], set activePageName to that value, save the file
  }
}

## Workflow

fn formatAuditReport(testRuns, exitCode) {
  // Build the human-friendly audit report from parsed JSON results and save it using the edit tool
  // Do NOT use bash heredoc or shell redirection — use `edit` tool to write the file
  // Use result.ParentDisplayName for page names and result.RuleName for rule names
  // For visual-level violations, result.Actual contains internal visual folder IDs

  AuditReport {
    sections {
      ExecutiveSummary {
        status = if (exitCode == 1) "❌ FAILED" else if (hasWarnings) "⚠️ PASSED WITH WARNINGS" else "✅ PASSED"
        metrics: { reportsAnalyzed, totalReports, criticalErrors, warnings, checksPassed }
        scope = match ($GITHUB_EVENT_NAME) {
          case "push" => "Incremental (changed reports only)"
          case "pull_request" => "Comprehensive (all reports)"
          default => "Comprehensive (all reports)"
        }
      }

      CriticalIssues {
        constraint: Group by category (Accessibility, UX, Performance, Design)
        constraint: Include fix instructions for each violation
        constraint: Use collapsible <details open> sections
      }

      Warnings {
        constraint: Use <details> (collapsed) for advisory findings
      }

      PassedChecks {
        constraint: Use <details> (collapsed)
      }

      QuickActionItems {
        prioritized list: 🔴 critical first, then 🟡 warnings
        format: "**[Report]** Fix description"
      }
    }

    formatting {
      markdown with tables
      emojis: ❌ errors, ⚠️ warnings, ✅ passed
      page names: "Display Name" (internalId)
    }
  }

  // Save the report using the edit tool
  edit("/tmp/gh-aw/agent/pbir-audit-report.md", auditReportContent)
  return auditReport
}

fn applyUxFixes(testRuns, workspace) {
  // Parse PBIInspector JSON results for UX-category violations only

  uxRules = [
    "FIRST_TAB_DEFAULT",
    "TOOLTIP_DRILLTHROUGH_HIDDEN",
    "NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_1",
    "NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_2",
    "SLICER_SEARCH_INPUT_SHOULD_BE_EMPTY"
  ]

  fixesApplied = 0
  fixRecords = []  // FixRecord[] — populated below, returned alongside fixesApplied

  for each result in testRuns.Results where result.Pass == false and result.RuleId in uxRules {
    ruleId = result.RuleId

    // Derive report folder: extract from TestedFilePath basename minus .pbip, append .Report
    // e.g. TestedFilePath ends in PbirRulesTest.pbip -> reportFolder = PbirRulesTest.Report
    reportFolder = basename(result.TestedFilePath).replace(".pbip", ".Report")

    // Extract page folder from ItemPath (last directory before page.json)
    // e.g. "ort\definition\pages\ENSURE_NON_DEFAULT_TITLES_FAIL\page.json" -> "ENSURE_NON_DEFAULT_TITLES_FAIL"
    pageFolder = extractPageFolder(result.ItemPath)

    // Reconstruct the full file path
    if (ruleId == "FIRST_TAB_DEFAULT") {
      filePath = workspace + "/" + reportFolder + "/definition/pages/pages.json"
    } else if (ruleId == "TOOLTIP_DRILLTHROUGH_HIDDEN") {
      // ItemPath is report-level ("ort") for this rule — Actual contains the page folder names
      // Iterate over each failing page in Actual and fix each one
      for each pageFolder in result.Actual {
        filePath = workspace + "/" + reportFolder + "/definition/pages/" + pageFolder + "/page.json"
        // Read and parse the page.json
        currentContent = cat(filePath)
        parsed = JSON.parse(currentContent)
        // Only apply fix if visibility property is missing or not already set correctly
        if (parsed.visibility != "HiddenInViewMode") {
          // Add visibility as a top-level property of the page JSON object
          parsed.visibility = "HiddenInViewMode"
          fixedContent = JSON.stringify(parsed, indent=2)
          edit(filePath, fixedContent)
          fixesApplied++
          echo "Fixed " + ruleId + " in " + filePath
          fixRecords.append({ ruleId: ruleId, filePath: reportFolder + "/definition/pages/" + pageFolder + "/page.json" })
        }
      }
      continue  // skip the generic fix block below
    } else {
      // Visual-level rules: use visual folder name from Actual[0]
      visualFolder = result.Actual[0]
      filePath = workspace + "/" + reportFolder + "/definition/pages/" + pageFolder + "/visuals/" + visualFolder + "/visual.json"
    }

    // Read the current file content
    currentContent = cat(filePath)

    // Apply the fix from UxFixPatterns[ruleId]
    fixedContent = applyFix(ruleId, currentContent, filePath)

    if (fixedContent != currentContent) {
      edit(filePath, fixedContent)
      fixesApplied++
      echo "Fixed " + ruleId + " in " + filePath
      // Record a workspace-relative path (strip leading workspace prefix)
      fixRecords.append({ ruleId: ruleId, filePath: filePath.removePrefix(workspace + "/") })
    }
  }

  return { fixesApplied: fixesApplied, fixRecords: fixRecords }
}

interface FixRecord {
  ruleId: string           // e.g. "FIRST_TAB_DEFAULT"
  issue: string            // one-line description of what was wrong
  fix: string              // one-line description of what was changed
  filePath: string         // workspace-relative path to the changed file (e.g. "PbirRulesTest.Report/definition/pages/pages.json")
}

// RuleDescriptions maps each UX ruleId to human-readable issue/fix text for the PR body
RuleDescriptions {
  FIRST_TAB_DEFAULT {
    issue: "The report's default page (`activePageName`) did not match the first entry in `pageOrder`."
    fix: "Updated `activePageName` in `pages.json` to match `pageOrder[0]`."
  }
  TOOLTIP_DRILLTHROUGH_HIDDEN {
    issue: "A Tooltip or Drillthrough page was visible to report consumers in view mode."
    fix: "Added `\"visibility\": \"HiddenInViewMode\"` to the page's `page.json`."
  }
  NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_1 {
    issue: "A slicer, bookmark navigator, or page navigator visual had visual headers enabled, adding unnecessary UI chrome."
    fix: "Added `visualHeader.show = false` to `visualContainerObjects` in the visual's `visual.json`."
  }
  NO_VISUAL_HEADERS_FOR_CERTAIN_VISUALS_2 {
    issue: "A decorative visual (shape, image, textbox, or action button) had visual headers enabled."
    fix: "Added `visualHeader.show = false` to `visualContainerObjects` in the visual's `visual.json`."
  }
  SLICER_SEARCH_INPUT_SHOULD_BE_EMPTY {
    issue: "A slicer had a pre-filled search term that would inadvertently filter data for report consumers."
    fix: "Removed the `selfFilter`/`selfFilterEnabled` flag and the pre-filled `filter` value from `visual.objects.general[0].properties` in the visual's `visual.json`."
  }
}

fn createPrIfNeeded(fixesApplied, fixRecords) {
  // fixRecords: FixRecord[] — one entry per file changed, collected during applyUxFixes
  if (fixesApplied > 0) {
    // Verify there are actual git changes before opening a PR
    diff = git("diff --quiet")
    if (diff indicates changes exist) {
      // Build a per-fix table for the PR body
      // GitHub file links use: https://github.com/{owner}/{repo}/blob/{branch}/{filePath}
      // Use git("branch --show-current") to get the branch name for the link
      branch = git("branch --show-current")
      repoUrl = "https://github.com/$GITHUB_REPOSITORY"

      fixRows = for each record in fixRecords:
        "| **$record.ruleId** | $RuleDescriptions[record.ruleId].issue | $RuleDescriptions[record.ruleId].fix | [$(record.filePath)]($repoUrl/blob/$branch/$record.filePath) |"

      prBody = """
This PR was automatically generated by the **PBIR Report Rules Checker** workflow.

## Summary

| Rule | Issue | Fix Applied | File Changed |
|---|---|---|---|
$fixRows

> ⚠️ Please review each linked file to confirm the change is correct before merging.
"""

      create-pull-request(
        title: "fix: auto-fix UX rule violations in PBIR reports",
        body: prBody
      )
    }
  }
}

// Main execution flow
workflow {
  inputs = read(Inputs)

  if (inputs.status contains "SKIP=true") {
    echo "No report changes detected. Skipping."
    exit(0)
  }

  // Step 1: Format and save the audit report
  report = formatAuditReport(inputs.testRuns, inputs.exitCode)

  // Step 2: Apply UX fixes if there are violations
  uxResult = applyUxFixes(inputs.testRuns, inputs.workspace)
  fixesApplied = uxResult.fixesApplied

  // Step 3: Create PR if fixes were applied
  createPrIfNeeded(fixesApplied, uxResult.fixRecords)

  // Step 4: Log summary
  echo "Status: " + report.ExecutiveSummary.status
  echo "Reports analyzed: " + report.ExecutiveSummary.metrics.reportsAnalyzed
  echo "Critical errors: " + report.ExecutiveSummary.metrics.criticalErrors
  echo "Warnings: " + report.ExecutiveSummary.metrics.warnings
  echo "UX fixes applied: " + fixesApplied

  exit(inputs.exitCode)
}

## Output Template

interface AuditReportFormat {
  title: "🔍 PBIR Report Rules Audit"
  metadata: { generated, status, event, scope }

  sections {
    ExecutiveSummary {
      metricsTable: { reportsAnalyzed, totalReports, criticalErrors, warnings, checksPassed, totalChecks }
      reportsOverview: table with { report, status, errors, warnings, pass }
    }

    CriticalIssues {
      constraint: Use <details open> for visibility
      categories: [Accessibility, UX, Performance, Design]
      for each violation: { report, page, visual, ruleId, howToFix }
      UX violations: note "Auto-fix applied via PR" if fix was applied
      include: Impact statement explaining why rule matters
    }

    Warnings {
      constraint: Use <details> (collapsed) to reduce noise
      same structure as CriticalIssues
    }

    PassedChecks {
      constraint: Use <details> (collapsed)
      list reports with clean audits
    }

    QuickActionItems {
      prioritized list: 🔴 critical first, then 🟡 warnings
      format: "**[Report]** Fix description"
      UX items: mark with 🔧 if auto-fixed
    }
  }

  formatting {
    markdown with tables
    emojis: ❌ errors, ⚠️ warnings, ✅ passed, 🔧 auto-fixed
    collapsible sections via <details>
    page names: "Display Name" (internalId)
  }
}

## Success Criteria

constraint PassingAudit {
  Audit report saved to /tmp/gh-aw/agent/pbir-audit-report.md using `edit` tool (NOT bash heredoc)
  UX violations fixed in PBIR JSON files using `edit` tool
  PR created with create-pull-request if any UX fixes were applied
  Report includes: executive summary, grouped findings, impact statements, quick actions
}
  
  Features {
    🔴 Critical issues grouped by impact area with remediation steps
    🟡 Warnings collapsed to reduce noise
    ✅ Passed checks collapsed to highlight problems
    📋 Quick action items prioritized by severity
  }
  
  constraint: Workflow fails (❌) if error-level violations found (blocks PR merge)
}

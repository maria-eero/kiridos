---
name: bug-escape-analysis
description: Use when asked to analyze customer-filed bugs from Jira, categorize them by root cause, generate a bug escape report with metrics, and identify trends. Requires the mcp-atlassian MCP server, GitHub MCP tools, and testrail-mcp server to be configured and enabled.
---

# Bug Escape Analysis Skill

Analyze customer-reported production bugs from Jira, categorize them by root cause and escape reason, populate a structured spreadsheet-style breakdown, generate a metrics summary table aligned to QA OKR tracking, and surface trends and repeat patterns.

## Agent

Use the `lead-quality-assurance-engineer` agent to execute this skill.

## Prerequisites

- **Jira MCP Server**: `mcp-atlassian` must be configured and enabled in Kiro MCP settings with valid Jira credentials.
- **GitHub MCP Tools**: GitHub MCP tools (`get_pull_request`, `get_pull_request_files`, `search_issues`) must be available for fetching PR descriptions and change details linked to Jira tickets.
- **Output Directory**: Reports are saved to `~/bug-escape-reports/`.
- **TestRail MCP Server**: `@tenbarrel6/testrail-mcp` must be configured and enabled in Kiro MCP settings with valid TestRail credentials (URL, username, API key). This enables automatic test case fetching and mapping to bugs.

## Inputs

Before generating the report, gather these from the user:

1. **Time period** — which quarter(s) to analyze (e.g., "Q1 2026", "Q4 2025 + Q1 2026")
2. **Team filter** (optional) — which team column to populate in the metrics table (WiFi, Device/eeroOS, Mobile, Web). If omitted, report covers all teams.
3. **Project filter** (optional) — override the default project list if needed. Defaults to: Connectivity, SWS, Core Engineering.
4. **TestRail project name** (optional) — the TestRail project to fetch test cases from. If not provided, the skill will call `get_projects` to list available projects and ask the user to pick one.

If the user says "run bug escape analysis" without specifying a period, ask for the quarter(s).

## Step 1: Fetch Bugs from Jira

Use the `jira_search` MCP tool with the following base JQL query. Adjust the `resolved` date range based on the user's requested time period.

### Base JQL Query

```
"Bug Source" = "Customer Feedback"
AND "Bug Source" NOT IN ("Dogfood Population", "Beta Population")
AND project in (Connectivity, "Software Services (SWS)", "Core Engineering")
AND status in (Done, "Dev Complete", "Dev Completed", Complete, Closed)
AND resolved >= startOfYear()
AND resolved < startOfYear(3)
AND issuetype in (Bug)
AND component not in ("enterprise-features", "wifi-ap-client", "wifi-internationalization")
AND "Team[Team]" not in (
  9ff21fc7-c17d-4ddb-a47a-acff31d8012b,
  831e646b-55a1-4d24-b87e-827f71e3d255-19,
  831e646b-55a1-4d24-b87e-827f71e3d255-11,
  877363e8-3ea9-4e90-adf9-91bfe7ce4942,
  14bfa1c4-143c-4da4-844e-809db30995dc
)
```

### Date Range Adjustment

Replace the `resolved` clauses based on the requested period:
- **Q1 2026**: `resolved >= "2026-01-01" AND resolved < "2026-04-01"`
- **Q2 2026**: `resolved >= "2026-04-01" AND resolved < "2026-07-01"`
- **Full year 2026**: `resolved >= "2026-01-01" AND resolved < "2027-01-01"`
- **Multi-quarter**: use the earliest start and latest end

### Fetching Process

1. Call `jira_search` with the adjusted JQL. Request these fields: `key`, `summary`, `priority`, `labels`, `components`, `status`, `creator`, `assignee`, `created`, `resolved`, `description`, `comment`, and all custom fields (Bug Source, Bug Source - Category, Team).
2. If results exceed the page limit, paginate until all issues are retrieved.
3. For each issue, call `jira_get_issue` to retrieve the full description, comments, and linked pull requests (dev panel / PR links).

## Step 2: Fetch Linked PR Descriptions from GitHub

For each Jira issue fetched in Step 1, extract linked pull request URLs from the issue's dev panel, description, or comments. Then use GitHub MCP tools to retrieve the full PR details.

### Process

1. **Extract PR links** — scan each issue's description, comments, and `jira_get_issue_development_info` results for GitHub PR URLs (patterns like `github.com/{owner}/{repo}/pull/{number}`).
2. **Fetch PR details** — for each linked PR, call `get_pull_request` with the extracted `owner`, `repo`, and `pull_number` to retrieve the PR title, body/description, and merge status.
3. **Fetch changed files** — for each linked PR, call `get_pull_request_files` to see which files were changed and the scope of the fix.
4. **Detect tests included in fix** — from the changed files list, identify any test files added or modified as part of the fix PR. Match files using these path/name patterns (case-insensitive):
   - Paths containing `tests/`, `test/`, `__tests__/`, `spec/`, or `specs/`
   - Filenames matching `test_*`, `*_test.*`, `*_spec.*`, `*Test.*`, `*Spec.*`
   - Filenames matching `conftest.py`, `fixtures.*`
   - Files with extensions `.test.ts`, `.test.js`, `.spec.ts`, `.spec.js`, `.test.py`

   For each PR, record:
   - **Tests included**: Yes / No
   - **Test files** (if Yes): list of test file paths added or modified
   - **Test type**: Unit / Integration / E2E (infer from path — e.g., `unit/` → Unit, `e2e/` or `integration/` → Integration/E2E, otherwise "Unknown")

5. **Store PR data per issue** — associate the fetched PR description(s), file change summaries, and test inclusion data with the corresponding Jira issue key for use in subsequent steps.

If a Jira issue has no linked PRs, note this — it may indicate the fix was a config change, operational action, or the ticket was closed as "Not a Bug."

## Step 3: Fetch Test Cases from TestRail

Use the TestRail MCP tools to fetch all test cases and build a mapping of test cases to Jira bugs.

### Discovery Process

1. **Identify the TestRail project** — if the user provided a project name, call `get_projects` and match by name. Otherwise, call `get_projects` to list all available projects and ask the user to select one.
2. **Get suites** — call `get_suites` with the project ID to list available test suites.
3. **Discover custom fields** — call `get_case_fields` to identify the field name for automation type (commonly `custom_automation_type` or similar). Look for a field whose label contains "automation" (case-insensitive).

### Suite Selection

4. **Default suites to search** — unless the user specifies otherwise, search only the following suites from the "eero SW QA" project:
   - **Mobile** (suite ID 9)
   - **Insight 2.0** (suite ID 79914)

   These are the primary suites where test cases for customer-facing app and web bugs are maintained. Other suites (eeroOS, Internet Backup, B2B, etc.) can be included if the user explicitly requests them or if a bug's component clearly maps to a different suite.

### Filtering: Only Match Code Change Bugs

5. **Before searching for TC matches, filter the bug list** — only attempt to find TestRail TC coverage for bugs whose Jira resolution is **"Code Changed"** (resolution ID 10306) or equivalent code-fix resolutions. Skip TC matching for bugs classified as:
   - Operational Change / Operational Gap
   - Not a Bug / WAD
   - Infrastructure / Service Issue
   - Data / Configuration Issue

   This avoids false "No TC" flags on bugs that are not testable software defects.

### Fetching Test Cases

6. **Fetch all test cases** — for each selected suite, call `get_cases` with the `project_id` and `suite_id`. The response includes each test case's `id`, `title`, `type_id`, `priority_id`, `refs` (Jira references), `section_id`, and custom fields (including the automation type field identified above).
7. **Paginate** — if the project has many test cases, `get_cases` may return paginated results. Continue fetching until all cases are retrieved.

### Building the Bug-to-TC Mapping

8. **Parse references** — for each test case, read the `refs` field (comma-separated Jira issue keys like `SWS-40691, SWS-41002`). Split by comma, trim whitespace, and extract keys matching `[A-Z]+-\d+`.
9. **Determine automation status** — read the automation type custom field for each TC. A test case is considered automated if the field value contains "Automated" (case-insensitive). All other values (Manual, Not Automated, None, null) mean manual.
10. **Build the lookup** — create a reverse mapping: `{ "SWS-40691": [{ id: "C2266830", title: "...", automated: true/false, type: "...", section: "..." }, ...] }`.

### Handling Edge Cases

- If the TestRail MCP server is not configured or unreachable, warn the user and fall back to inference mode for all bugs.
- If a bug key from Jira doesn't appear in any TC's `refs`, that bug has no mapped test cases (likely O1.4).
- If a bug key maps to multiple TCs, list all of them. A bug is considered "has automated coverage" if **at least one** mapped TC is automated.

## Step 4: Categorize Each Bug

For every fetched bug, extract and populate the following columns. Use the ticket description, fields, comments, **linked PR descriptions/changed files from Step 2**, and **TestRail TC mapping from Step 3 (if available)** to fill each column.

### Column Definitions

| Column | How to Populate |
|--------|----------------|
| **Issue Type** | Always "Bug" (from JQL filter) |
| **Key** | Jira issue key (e.g., SWS-40691) |
| **Summary** | Issue summary field |
| **Priority** | Priority field (P0–P3) |
| **Labels** | All labels, semicolon-separated |
| **Components** | All components, semicolon-separated |
| **Status** | Current status |
| **Flagged** | Flagged field value if present |
| **Creator** | Issue creator display name |
| **Assignee** | Issue assignee display name |
| **QA Assignee** | Custom field if available; otherwise leave blank |
| **Affects versions** | Affects versions field |
| **Bug Source** | Custom field "Bug Source" |
| **Bug Source - Category** | Custom field "Bug Source - Category" |
| **Created** | Created date |
| **Root cause** | Synthesize from description + comments + **PR description and changed files from Step 2**. Write a concise root cause statement (1–3 sentences). If the ticket is not a real bug, state that clearly (e.g., "Not a bug — working as designed"). |
| **How to test** | Extract or synthesize test steps from description/comments/**PR description from Step 2**. PR descriptions often include reproduction steps or testing notes. If operational gap, write "N/A — Manual operational request." |
| **Test Case Exists** | Check the bug-to-TC mapping from Step 3. If the bug key has mapped TCs, "Yes" (include TC IDs, e.g., "Yes — C2266830, C2267001"). If no mapped TCs, "No". If TCs exist but none cover the exact failure scenario described in the bug, "Partially". If TestRail was unreachable, fall back to extracting from ticket fields/comments. |
| **Automation Status** | For each mapped TC from Step 3, state whether it is automated or manual based on the automation type field. Summarize as: "Automated (C2266830), Manual (C2267001)" or "All Manual" / "All Automated" / "Mixed". If TestRail was unreachable, infer from ticket content. |
| **Can it be covered by automation** | Extract from ticket. Values: Yes, No, Yes with low ROI |
| **Can it be covered by manual test** | Extract from ticket. Values: Yes, No |
| **Tests Included in Fix** | From Step 2 test detection. Values: "Yes — [file list]" if test files were added/modified in the fix PR, "No" if the PR contained no test files, "N/A" if no linked PR exists. This indicates whether the developer shipped regression prevention alongside the fix. |
| **Why we miss this** | Infer from root cause + test case existence + comments + **PR changed files scope from Step 2** (e.g., if the fix touched files outside the component's usual test coverage) |
| **Next** | Extract recommended next action from comments (e.g., "Add TC", "Add additional steps to C2266830") |
| **Accepted by Product** | Extract from comments if mentioned |
| **Comments** | Key reviewer/QA comments relevant to the escape analysis |
| **Reviewer** | Reviewer name if mentioned in comments |

### Bug Classification Categories

Classify each bug into exactly one primary category:

1. **Code Bug** — actual software defect (logic error, regression, data issue)
2. **Not a Bug / WAD** — working as designed, third-party issue, or user error
3. **Operational Gap / Missing Feature** — no self-service capability exists; requires manual backend intervention
4. **Infrastructure / Service Issue** — third-party service outage, DNS issue, deployment problem
5. **Data / Configuration Issue** — misconfigured org, stale data, provisioning failure

### O1.x Root Cause Auto-Derivation

For bugs classified as "Code Bug", auto-derive the O1.x escape reason using the real test case mapping from Step 3. If TestRail was unreachable, fall back to inferring from ticket content.

| O1.x | Condition (with TestRail data) | Condition (fallback — no TestRail) |
|------|-------------------------------|-------------------------------------|
| **O1.1** — Test case exists, manual only, not executed | Bug has mapped TCs from Step 3 AND **all** mapped TCs have automation type = Manual/Not Automated | `Test Case Exists` = Yes/Partially AND `Can it be covered by automation` = Yes AND no evidence of automated test running |
| **O1.2** — Test case exists, automated but flaky/skipped/blocked | Bug has mapped TCs from Step 3 AND **at least one** mapped TC is Automated AND evidence from Jira comments/description that the automated test was flaky, skipped, or blocked | `Test Case Exists` = Yes AND evidence of automated test that was flaky, skipped, or blocked |
| **O1.3** — QA was not engaged | Comments/description indicate QA was not involved in the feature/change that caused the bug | Comments/description indicate QA was not involved in the feature/change that caused the bug |
| **O1.4** — No test case | Bug key has **no mapped TCs** in the Step 3 mapping | `Test Case Exists` = No AND `Next` suggests adding a new TC |
| **O1.5** — Other | Does not fit O1.1–O1.4 (e.g., cross-functional gap, environment-specific, timing issue) | Does not fit O1.1–O1.4 |

If a bug cannot be confidently classified, assign **O1.5** and add a flag: `⚠️ Needs manual review` in the Comments column.

## Step 5: Identify Repeat Patterns

After categorizing all bugs, scan for repeat patterns:

1. **Group by Component** — find components with 3+ bugs
2. **Group by Root Cause similarity** — find bugs with the same underlying issue (e.g., all PI admin transfer requests, all NextJS migration regressions)
3. **Group by "Not a Bug"** — count false escalations

Build a **Repeat Offenders** table:

```markdown
### Repeat Offenders / Recurring Patterns

| Pattern | Count | Category | Example Tickets | Recommendation |
|---------|-------|----------|----------------|----------------|
| [pattern description] | [n] | [category] | [ticket keys] | [actionable recommendation] |
```

## Step 6: Generate Metrics Summary Table

Populate the QA OKR metrics table for the requested period. Filter counts by the user's team filter if provided; otherwise show totals across all teams.

```markdown
### Bug Escape Metrics — [Period]

|  | Metric | WiFi | Device / eeroOS | Mobile | Web | Total |
|:---|:---|:---|:---|:---|:---|:---|
| O1 | Total P0/P1 production escapes/escalations/incidents (quarterly update) | | | | | [count] |
| O1.1 | Root-cause: Test case exists — Manual Only (no automated test) and not executed | | | | | [count] |
| O1.2 | Root-cause: Test case exists — Automated test case but flaky/skipped/blocked | | | | | [count] |
| O1.3 | Root-cause: QA was not engaged | | | | | [count] |
| O1.4 | Root-cause: No test case — Automated or Manual | | | | | [count] |
| O1.5 | Root-cause: Other (e.g., cross-functional gaps) | | | | | [count] |
| O2 | % of P0/P1 regression suite automated and passing on Toaster/pytest-eero | *manual* | *manual* | *manual* | *manual* | *manual* |
| O3 | Total % P0/P1 test cases automated by feature launch | *manual* | *manual* | *manual* | *manual* | *manual* |
| I1 | Total tests automated (non-flaky test cases only) | *manual* | *manual* | *manual* | *manual* | *manual* |
| I1.1 | Total tests automated (non-flaky test cases only) — human-authored | *manual* | *manual* | *manual* | *manual* | *manual* |
| I1.2 | Total tests automated (non-flaky test cases only) — AI-authored (Kiro/Cline/drones) | *manual* | *manual* | *manual* | *manual* | *manual* |
| I2 | # of P0/P1 test cases marked automatable and not yet automated | *manual* | *manual* | *manual* | *manual* | *manual* |

**When TestRail data is available from Step 3**, the following cells can be populated automatically instead of being left as *manual*:
- **I2**: count mapped TCs where automation type ≠ Automated and the linked bug is P0/P1. This gives a lower bound — TCs not linked to bugs won't be counted.
| I3 | Escapee-to-automated-test closure time (days from escape to TC authored) | | | | | [avg] |
```

Cells marked *manual* require data from CI dashboards / test repos — not derivable from Jira alone. When TestRail data is available, some of these cells are populated automatically (see note above). Leave remaining *manual* cells for the user to fill when creating the Google Doc.

### Counting Rules

- **O1 total**: count only P0 and P1 bugs classified as "Code Bug". Exclude "Not a Bug / WAD", "Operational Gap", and "Infrastructure / Service Issue".
- **O1.1–O1.5**: sub-counts of O1 by auto-derived root cause. Must sum to O1.
- **I3**: for bugs where `Next` indicates a TC was added and a resolution date exists, calculate days between bug creation and TC closure. If data is insufficient, mark as "Insufficient data".
- **Not a Bug count**: report separately below the table as: `False escalations (Not a Bug / WAD): [count]`
- **Operational Gap count**: report separately as: `Operational gaps (missing feature): [count]`

## Step 7: Trend Analysis

When the user provides data for multiple quarters, generate trend analysis:

### Quarter-over-Quarter Comparison

```markdown
### Trend: Bug Escapes by Quarter

| Metric | [Q-1] | [Q] | Delta | Trend |
|--------|-------|-----|-------|-------|
| Total P0/P1 escapes | [n] | [n] | [+/-n] | ↑/↓/→ |
| O1.1 (manual only, not executed) | [n] | [n] | [+/-n] | ↑/↓/→ |
| O1.4 (no test case) | [n] | [n] | [+/-n] | ↑/↓/→ |
| False escalations | [n] | [n] | [+/-n] | ↑/↓/→ |
| Operational gaps | [n] | [n] | [+/-n] | ↑/↓/→ |
```

### Category Distribution Shift

Show how the proportion of each bug classification category changed between periods.

### Emerging Patterns

Identify:
- **New components** appearing in escapes that weren't present in the prior period
- **Resolved patterns** — repeat offenders from the prior period that no longer appear
- **Worsening patterns** — repeat offenders with increasing count
- **Recommendations** — top 3 actionable items based on the trend data (e.g., "PI admin self-service would eliminate ~12 escalations/quarter", "NextJS migration needs dedicated regression suite")

## Output Format

Generate a single markdown report saved to `~/bug-escape-reports/bug-escape-report-[period].md` with these sections:

```
# Bug Escape Analysis Report — [Period]

## Executive Summary
[2–3 sentence summary: total bugs analyzed, key finding, top recommendation]

## Metrics Summary
[O1.x table from Step 6]
[False escalation and operational gap counts]

## Repeat Offenders
[Table from Step 5]

## Trend Analysis
[From Step 7 — only if multi-quarter data is available]

## Detailed Bug Breakdown
[Full spreadsheet-style table from Step 4, sorted by Priority then Created date]

## Classification Flags
[List any bugs flagged with ⚠️ Needs manual review, with the reason]

## TestRail Coverage Summary
[Show: total TCs fetched from TestRail, bugs with TC coverage vs without, automated vs manual TC breakdown, and list of bugs with no TC coverage. If TestRail was unreachable, note that and explain fallback.]

## Regression Prevention Coverage
[Summary of whether fix PRs included tests to prevent recurrence]

### Summary
- Bugs with tests included in fix PR: [count] / [total code bugs] ([%])
- Bugs with NO tests in fix PR: [count] / [total code bugs] ([%])
- Bugs with no linked PR: [count]

### Bugs Missing Tests in Fix PR
[Table of code bugs where the fix PR did NOT include test files — these are candidates for follow-up test authoring]

| Key | Summary | Fix PR | Files Changed | Tests Included | Recommendation |
|-----|---------|--------|---------------|----------------|----------------|
| [key] | [summary] | [PR link] | [count] | No | [e.g., "Add unit test for edge case"] |

## Appendix: JQL Query Used
[The exact JQL query executed]
```

### Report Naming Convention

- Single quarter: `bug-escape-report-Q1-2026.md`
- Multi-quarter: `bug-escape-report-Q4-2025-Q1-2026.md`
- Full year: `bug-escape-report-2026.md`

## Critical Rule: Always Save Report to File

**Every execution of this skill MUST write the full report to a markdown file in `~/bug-escape-reports/` before presenting any results in chat.** Never display results only in chat without saving the file first. The user needs the persisted file to generate Google Docs.

After saving, confirm the file path to the user.

## Workflow Summary

1. User requests bug escape analysis for a time period
2. Skill fetches bugs via `jira_search` + `jira_get_issue` MCP tools
3. Skill fetches linked PR descriptions and changed files via `jira_get_issue_development_info` + `get_pull_request` + `get_pull_request_files` GitHub MCP tools
4. Skill fetches test cases from TestRail via `get_projects` + `get_suites` + `get_case_fields` + `get_cases` MCP tools, builds a bug-to-TC mapping with automation status
5. Skill categorizes each bug (columns + classification + O1.x), using PR data and **TestRail TC mapping** to fill root cause, test case existence, automation status, and why we miss this
6. Skill identifies repeat patterns
7. Skill generates metrics summary table (with TestRail-derived data)
8. Skill runs trend analysis (if multi-quarter)
9. **Skill writes the FULL report to `~/bug-escape-reports/bug-escape-report-[period].md`**
10. Skill confirms the saved file path to the user
11. Skill presents the executive summary and metrics table in chat, and notes any ⚠️ flags for manual review

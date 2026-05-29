# Bug Escape Analysis — Kiro Skill

## What is a Bug Escape?

A **bug escape** is a software defect that was not caught during development or QA and reached production, where it was discovered by a customer. Bug escape analysis is the process of examining these escaped bugs to understand *why* they weren't caught and *what can be done* to prevent similar escapes in the future.

This skill automates that analysis by pulling customer-reported bugs from Jira, categorizing them by root cause, and generating a structured report with metrics aligned to our QA OKR tracking.

## What This Skill Does

When you ask Kiro to "run bug escape analysis for Q1 2026 for the SWS project," it:

1. **Fetches bugs from Jira** — queries for customer-reported bugs (Bug Source = "Customer Feedback") that were resolved in the requested time period
2. **Fetches linked PR descriptions from GitHub** — pulls PR details for bugs that have linked pull requests, to better understand the code change and root cause
3. **Maps test cases from TestRail** — fetches test cases directly from TestRail via MCP, maps them to bugs using Jira references, and determines whether each test case is automated or manual
4. **Categorizes each bug** into one of five classifications:
   - **Code Bug** — actual software defect (logic error, regression, data issue)
   - **Operational Gap** — no self-service capability exists; required manual backend intervention
   - **Not a Bug / WAD** — working as designed, third-party issue, or user error
   - **Data / Configuration Issue** — misconfigured org, stale data, provisioning failure
   - **False Escalation** — cannot reproduce, insufficient info, or ISP-side issue
5. **Assigns O1.x escape codes** to Code Bugs, explaining *why* the bug escaped QA:
   - **O1.1** — test case existed but wasn't executed
   - **O1.2** — automated test existed but was flaky/skipped/blocked
   - **O1.3** — QA was not engaged in the feature/change
   - **O1.4** — no test case existed for the scenario
   - **O1.5** — other (cross-functional gap, environment-specific)
6. **Identifies repeat patterns** — groups bugs by component, root cause similarity, and recurring themes
7. **Generates a metrics summary** aligned to the QA OKR table (O1 total, O1.1–O1.5 breakdown)
8. **Writes a full markdown report** to `~/bug-escape-reports/`

## How to Use It

### Prerequisites

- **Jira MCP server** (`mcp-atlassian`) must be configured in your Kiro MCP settings with valid Jira credentials
- **GitHub MCP tools** should be available for fetching PR descriptions (optional — the skill works without them but classifications may be less precise)
- **TestRail MCP server** (`@tenbarrel6/testrail-mcp`) must be configured in your Kiro MCP settings with valid TestRail credentials. This enables automatic test case fetching — no manual CSV export needed. See [TestRail MCP Setup](#testrail-mcp-setup) below.

### Running the Skill

Tell Kiro:

> "Run bug escape analysis for Q1 2026 for the SWS project"

The skill will automatically fetch test cases from TestRail via MCP — no manual export needed.

You can customize:
- **Time period**: any quarter, multi-quarter range, or full year
- **Project filter**: SWS, Connectivity, Core Engineering, or all three (the default)

### Output

The skill writes a markdown report to `~/bug-escape-reports/` with this structure:

```
bug-escape-report-SWS-Q1-2026.md
├── Executive Summary
├── Metrics Summary (classification breakdown + O1.x table)
├── Repeat Offenders / Pattern Clusters
├── Detailed Bug Breakdown (by classification)
├── Classification Flags & Edge Cases
├── Recommendations (immediate + medium-term)
└── Appendix (JQL query, O1.x reference, methodology)
```

## How the JQL Query Works

The base query in the skill file targets bugs that:
- Were reported by customers (`"Bug Source" = "Customer Feedback"`)
- Exclude internal testing populations (dogfood, beta)
- Are resolved/closed within the requested date range
- Are typed as Bug

The query also includes component and team exclusion filters that were originally tuned for multi-project analysis. **When running for a single project like SWS, you may want to use the broader query without those exclusions** to capture the full picture — discuss with Kiro when running.

## How O1 Metrics Work

Only **P0 and P1 Code Bugs** count toward the O1 escape metric. This is intentional:
- Operational gaps, false escalations, and not-a-bug tickets are tracked for trend analysis but don't represent QA escapes
- P2/P3 bugs are listed for completeness but excluded from O1 counts
- The O1.1–O1.5 sub-codes must sum to the O1 total

## What to Do With the Report

1. **Review classification flags** — some bugs are ambiguous and may need manual reclassification
2. **Fill in manual metrics** — O2, O3, I1, I2 require data from TestRail/CI dashboards that Jira doesn't have
3. **Use repeat offender patterns** to prioritize QA investments (e.g., "11 PI admin requests → build self-service tooling")
4. **Transfer to Google Docs** for team review and OKR tracking

## TestRail MCP Setup

The skill fetches test cases directly from TestRail using the `@tenbarrel6/testrail-mcp` MCP server. No manual CSV export is needed.

### 1. Get Your TestRail API Key

1. Log in to your TestRail instance
2. Go to **My Settings** (click your name in the top right)
3. Navigate to the **API Keys** tab
4. Click **Add Key** to generate a new API key
5. Copy and save the key securely

### 2. Add to Kiro MCP Settings

Add the following to your `~/.kiro/settings/mcp.json`:

```json
"testrail": {
  "command": "npx",
  "args": ["-y", "@tenbarrel6/testrail-mcp"],
  "env": {
    "TESTRAIL_URL": "https://your-company.testrail.io",
    "TESTRAIL_USERNAME": "your-email@company.com",
    "TESTRAIL_API_KEY": "your-api-key-here"
  },
  "disabled": false,
  "autoApprove": []
}
```

### How It Works

When the skill runs, it:

1. Calls `get_projects` to find the TestRail project
2. Calls `get_suites` to list test suites in that project
3. Calls `get_case_fields` to discover the automation type custom field
4. Calls `get_cases` for each suite to fetch all test cases with their `refs` (Jira keys) and automation status
5. Builds a reverse lookup mapping each bug to its covering test cases
6. Uses this real data to populate **Test Case Exists**, **Automation Status**, and drive **O1.x classification**

### Impact on Report Accuracy

| Aspect | Without TestRail | With TestRail MCP |
|--------|-----------------|-------------------|
| Test Case Exists | Inferred from ticket text | Real data from TestRail |
| Automation Status | Unknown or guessed | Exact per-TC status |
| O1.1 vs O1.4 classification | Often defaults to O1.4 | Correctly distinguishes "has manual TC" (O1.1) from "no TC" (O1.4) |
| I2 metric | Fully manual | Partially populated from TestRail |

## Known Limitations

- **GitHub PR access**: if your PAT doesn't have access to the org's private repos, PR descriptions won't be fetched. The skill notes this in the report and relies on Jira comments alone for root cause analysis.
- **O1.x accuracy**: the TestRail MCP integration provides high-accuracy O1.x classification by using real test case data. If the TestRail MCP server is not configured or unreachable, the skill falls back to inferring from Jira comments, which typically defaults to O1.4 (no test case) when information is ambiguous.
- **Manual metrics**: O2, O3, I1, I3 cells require data from CI dashboards that aren't accessible via Jira or TestRail. I2 can be partially populated from TestRail data.
- **TestRail coverage**: the mapping only works for bugs whose Jira keys appear in the `refs` field of test cases. If your team doesn't consistently link TCs to Jira tickets in TestRail, the mapping will be incomplete.

## File Structure

```
~/.kiro/skills/bug-escape-analysis/
├── SKILL.md    # The skill definition (Kiro reads this)
├── README.md   # This file (for humans)
```

```
~/bug-escape-reports/
├── bug-escape-report-Q1-2026.md        # Core Engineering report
├── bug-escape-report-SWS-Q1-2026.md    # SWS report
└── ...                                  # Future reports
```

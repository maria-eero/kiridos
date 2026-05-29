# Bug Escape Analysis Skill

Analyzes customer-reported production bugs from Jira, categorizes them by root cause, generates a metrics report aligned to QA OKRs, and surfaces trends and repeat patterns.

## Prerequisites

- Jira MCP (`mcp-atlassian`) configured
- GitHub MCP tools enabled
- TestRail MCP (`@tenbarrel6/testrail-mcp`) configured

## Artifacts

| # | Artifact | Format |
|---|----------|--------|
| 1 | Bug Escape Report (detailed breakdown + metrics) | Markdown file |

Saved to `~/bug-escape-reports/bug-escape-report-[period].md`

## How It Works

1. Fetches bugs from Jira using a predefined JQL (Customer Feedback source, resolved in period)
2. Fetches linked PR descriptions and changed files from GitHub
3. Fetches test cases from TestRail and builds a bug-to-TC mapping
4. Categorizes each bug: root cause, O1.x escape reason, test coverage, automation status
5. Identifies repeat patterns and recurring offenders
6. Generates OKR metrics table (O1–O1.5, I-metrics)
7. Runs trend analysis (if multi-quarter)
8. Saves full report to file

## Usage

```
Run bug escape analysis for Q1 2026, filtered to the Mobile team.
```

## Report Sections

- **Executive Summary** — key finding + top recommendation
- **Metrics Summary** — O1.x table with team breakdown
- **Repeat Offenders** — recurring patterns with recommendations
- **Trend Analysis** — quarter-over-quarter comparison (multi-quarter only)
- **Detailed Bug Breakdown** — full spreadsheet-style table per bug
- **TestRail Coverage Summary** — TC mapping stats
- **Regression Prevention Coverage** — whether fix PRs included tests

## Key Features

- Auto-derives O1.x root cause using real TestRail TC mapping
- Detects if fix PRs included test files (regression prevention signal)
- Supports multi-quarter trend analysis
- Classifies bugs: Code Bug, Not a Bug/WAD, Operational Gap, Infrastructure, Data/Config

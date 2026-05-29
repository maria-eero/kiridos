# Sprint Bug Health Report Skill

Generates a sprint bug health report with metrics, trend indicators, and actionable recommendations. Published directly to Confluence.

## Prerequisites

- Jira MCP (`mcp-atlassian`) configured for Core Engineering / SWS projects
- Confluence MCP (`mcp-atlassian`) configured with write access to QA space

## Artifacts

| # | Artifact | Format |
|---|----------|--------|
| 1 | Defect Management Report | Markdown + Confluence page |
| 2 | Visual Dashboard (PNG) | Generated via dashboard script |

Saved to `~/sprint-bug-reports/defect-management-report-[team]-[sprint-id]-[date].md`

## How It Works

1. Asks for project (Core Engineering or SWS), team, and sprint ID
2. Fetches current + previous sprint bugs from Jira
3. QAE agent analyzes: severity, components, platforms, epics, bug source/reason, resolution rate
4. Sub-classifies "Incorrect Logic" bugs via GitHub PR diff analysis
5. SDM agent produces health score, recommendations, and carry-over assessment
6. Generates visual dashboard PNG
7. Publishes to Confluence with TOC, JQL hyperlinks, and dashboard image

## Usage

```
Generate a sprint bug health report for Core Engineering, Onboarding & Insight team, sprint ID 15981. Previous sprint ID is 15432.
```

## Report Sections

- **Sprint Health** — 🔴/🟡/🟢 score with key metrics
- **Severity Distribution** — priority + customer impact, with trend
- **Component & Platform Hotspots** — top areas generating bugs
- **Bugs by Epic** — which features drive defect volume
- **Bug Source & Classification** — who found it, root cause breakdown
- **Resolution Metrics** — resolved vs carried over
- **Operational Excellence Highlights** — op-ex day contributions
- **Sprint-over-Sprint Trend** — delta comparison with highlights
- **Decisions & Recommendations** — top 3 actions for next sprint

## Key Features

- Every table includes previous sprint comparison with trend arrows
- All counts are JQL hyperlinks (click to see exact tickets in Jira)
- "Incorrect Logic" sub-classification via actual PR diff analysis
- Dual-agent approach: QAE for data, SDM for strategic recommendations

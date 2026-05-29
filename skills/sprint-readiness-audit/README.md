# Sprint Readiness Audit Skill

Audits a newly created sprint for gaps that affect SDLC flow — missing story points, empty QA assignments, unfilled bug metadata, and missing descriptions. Produces a concise, actionable report for the SDM.

## Prerequisites

- Jira MCP (`mcp-atlassian`) configured with access to the target project

## Artifacts

| # | Artifact | Format |
|---|----------|--------|
| 1 | Sprint Readiness Audit Report | Markdown (chat + optional Confluence) |

Optionally saved to `~/sprint-readiness-reports/` or published to Confluence.

## How It Works

1. Asks for the Sprint ID
2. Fetches all sprint tickets from Jira
3. Three-agent pipeline runs gap detection:
   - **Program Manager** — scope clarity, delivery risk, process gap patterns
   - **Lead QAE** — QA coverage risk, bug metadata health, testability flags
   - **SDM** — synthesizes into prioritized action plan with health score
4. Generates report ordered by severity

## Usage

```
Audit sprint 16042 for readiness gaps.
```

## Gaps Detected

| Gap | Severity |
|-----|----------|
| Missing Story Points | 🔴 Critical |
| Missing Description | 🔴 Critical |
| Empty QA Assignment | 🟡 High |
| Empty Bug Reason | 🟡 High |
| Empty Bug Source | 🟡 High |
| No Components | 🟠 Medium |
| Unassigned Ticket | 🟠 Medium |

## Report Sections

- **Sprint Health** — 🔴/🟡/🟢 score
- **Gap Summary** — counts and percentages per gap type
- **Critical Gaps** — ticket-level detail for immediate action
- **High Gaps** — QA and bug metadata issues
- **Decisions & Actions** — top 5 prioritized actions with suggested owners

## Health Score Thresholds

- 🔴 Red: >25% tickets have critical gaps
- 🟡 Yellow: 10–25% critical OR >40% high gaps
- 🟢 Green: <10% critical AND <25% high gaps

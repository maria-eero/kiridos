# Quality Report Skill

Generates a comprehensive quality report for a feature — covering test coverage, risk assessment, bug metrics, and release readiness with per-scope QA sign-off.

## Prerequisites

- Jira MCP (`mcp-atlassian`) configured
- TestRail MCP (optional, enhances coverage data)
- GitHub MCP (optional, for code churn analysis)

## Artifacts

| # | Artifact | Format |
|---|----------|--------|
| 1 | Quality Report (1-pager, Confluence-ready) | Markdown file |

Saved to `~/quality-reports/quality-report-[feature]-[date].md`

## How It Works

1. Gathers feature context from Jira epic(s) — stories, bugs, status
2. Assesses test coverage from TestRail (or available data)
3. Consolidates bug metrics (pre-prod vs escaped, severity, resolution rate, age)
4. Evaluates risk across 6 dimensions
5. Identifies post-launch risks
6. Determines QA sign-off verdict per scope (🟢 GO / 🟡 GO with Risks / 🔴 NO-GO)
7. Generates conditions for sign-off
8. Saves report to file

## Usage

```
Generate a quality report for the Network Topology feature.
Epic: CORE-1234
Scopes: iOS, Android
Target release: 6.60
Release stage: Beta only
Test plan: [Confluence link]
```

## Required Inputs

- Feature/project name
- Jira epic key(s)
- Scopes to evaluate (e.g., Android, iOS)
- Target release version/date
- Release stage (Production or Beta/Staging)
- Test plan document link

## Report Sections

- **QA Sign-off** — per-scope verdict with justification
- **Blocking Issues** — open P0/P1 bugs
- **Development Status** — story completion by scope
- **Bug Metrics** — status, severity, resolution rate, age, assignee breakdown
- **Risks** — coverage gaps, code churn, dependencies, platform, automation debt
- **Conditions for Sign-off** — what must be resolved, what carries risk, future observations

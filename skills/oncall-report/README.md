# On-Call Report Generator Skill

Generates the automation on-call handoff report by collecting pipeline data from GitLab, calculating pass rates, pulling related Jira tickets and PRs, then publishes the report to Confluence.

## Prerequisites

- `glab` CLI configured with access to the GitLab project
- Confluence MCP (`mcp-atlassian`) configured for the `QA` space
- Jira MCP (`mcp-atlassian`) configured for the `QA` project
- GitHub CLI (`gh`) configured for PR data

## Artifacts

| # | Artifact | Format |
|---|----------|--------|
| 1 | On-Call Handoff Report | Confluence page |
| 2 | Slack Handoff Message | Ready-to-paste text |

Published to Confluence under `QA > On-call Reports` (parent page ID: `5475172360`).

## How It Works

1. Collects GitLab pipeline data for the date range (main branch scheduled runs)
2. Classifies pipelines by suite (Sanity ~51–54 tests, Smoke Mobile ~150–156 tests)
3. Calculates pass rates per day and overall (excluding skipped tests)
4. Determines report type based on pass rate threshold:
   - **≥95%** → Routine Monitoring Form
   - **<95%** → Maintenance Triage Form
5. Pulls merged and open PRs from GitHub
6. Queries Jira for related tickets (maintenance epic, appium-bug label)
7. Asks for supplemental info (handoff notes, flaky tests, environment issues)
8. Publishes to Confluence
9. Generates a Slack handoff message

## Usage

```
Generate the on-call report for this week. GitLab project: group/project-name. Filter for my pipelines (maria/) and PRs (maria-eero).
```

## Report Types

### Routine Monitoring (≥95% pass rate)
- Overview, test suite pass rates, pipelines monitored
- Minor issues (flaky tests, environment warnings)
- Handoff notes with action items for next shift

### Maintenance Triage (<95% pass rate)
- Detection & pass rate check with trend classification
- GitLab failure investigation (failed jobs, error analysis, root cause)
- Health score assessment
- Severity classification (P1–P4)
- Resolution actions and verification

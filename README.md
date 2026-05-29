# kiridos

Custom Kiro CLI skills and agents for QA automation — bug analysis, sprint reporting, quality assessments, and on-call handoffs.

## Skills

| Skill | Description |
|-------|-------------|
| [bug-escape-analysis](skills/bug-escape-analysis/) | Analyzes customer-reported production bugs, categorizes by root cause (O1.x), generates OKR metrics and trend reports |
| [oncall-report](skills/oncall-report/) | Generates on-call handoff reports from GitLab pipelines, GitHub PRs, and Jira tickets; publishes to Confluence |
| [qa-process-planning](skills/qa-process-planning/) | Generates QA process flowchart, test plan, and automation plan from project documentation |
| [quality-report](skills/quality-report/) | Produces a release readiness report with test coverage, bug metrics, risk assessment, and per-scope QA sign-off |
| [sprint-bug-health-report](skills/sprint-bug-health-report/) | Sprint defect management report with severity trends, component hotspots, and recommendations; publishes to Confluence |
| [sprint-readiness-audit](skills/sprint-readiness-audit/) | Audits a new sprint for missing story points, QA assignments, bug metadata, and descriptions |

## Agents

| Agent | Role |
|-------|------|
| [lead-quality-assurance-engineer](agents/lead-quality-assurance-engineer/) | Lead QAE (QAE III) — test strategy, coverage analysis, bug categorization |
| [lead-software-developer-in-test](agents/lead-software-developer-in-test/) | Lead SDET (SDE-T III) — test architecture, framework design, automation strategy |
| [program-manager-iii](agents/program-manager-iii/) | PM III — scope clarity, risk management, stakeholder alignment |
| [software-developer-manager-iii](agents/software-developer-manager-iii/) | SDM III — feasibility, resourcing, prioritized action plans |
| [sde-iii](agents/sde-iii/) | SDE III — system architecture, code quality, operational excellence |

## Prerequisites

- [Kiro CLI](https://kiro.dev) installed
- MCP servers configured:
  - `mcp-atlassian` (Jira + Confluence)
  - `@tenbarrel6/testrail-mcp` (TestRail)
  - GitHub MCP tools
- `glab` CLI (for oncall-report)
- `gh` CLI (for PR data)

## Installation

Copy the contents into your `~/.kiro/` directory:

```bash
cp -r skills/* ~/.kiro/skills/
cp -r agents/* ~/.kiro/agents/
```

## How It Works

Skills are invoked via natural language prompts in Kiro CLI. Each skill uses one or more specialized agents in a pipeline to produce structured artifacts (reports, plans, audits) that are saved locally and/or published to Confluence.

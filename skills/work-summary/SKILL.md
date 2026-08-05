---
name: work-summary
description: Generate a Work Summary for talent review. Pulls activity data from GitHub, Amazon Code Browser, JIRA, and TestRail, then drafts a < 2-page summary in the official template format aligned to QAE role guidelines. Supports interactive curation of contributions before final output.
---

# Work Summary Skill

End-to-end skill that gathers engineering activity data, presents it for curation, and produces a polished Work Summary document in the official eero/Amazon template format.

## Invocation

> "Generate my work summary for H1 2026"
> "Create my work summary for {period}"

## Required Inputs

- **period**: The time range (e.g., `H1 2026`, `H2 2025`, `2025`). Defaults to H1 of the current year.
- **github_username**: Developer's GitHub username in eero-inc org (default: `henrique-eero`)
- **amazon_username**: Developer's Amazon alias (default: `prohenri`)
- **testrail_email**: TestRail login email (default: `henrique@eero.com`)
- **role_level**: QAE level for role guideline alignment (default: `QAE II`)

## Optional Inputs

- **jira_name**: Display name in JIRA (default: `Henrique Rodrigues`)
- **projects_context**: Free-text descriptions of projects/contributions that lack code artifacts (e.g., process improvements, cross-team initiatives, mentoring)
- **focus_areas**: Specific contributions the user wants highlighted

## Process

### Phase 1: Data Gathering

Execute all data sources in parallel where possible.

#### 1A. GitHub Activity (eero-inc org)

```bash
# Find all repos contributed to in period
gh search commits --author={github_username} --owner=eero-inc --committer-date={start}..{end} --limit 100 --json repository | jq -r '[.[].repository.name] | unique | .[]'

# For each repo, get PRs authored
gh pr list --repo eero-inc/{repo} --author {github_username} --state merged --search "merged:{start}..{end}" --limit 100 --json number,title,body,mergedAt

# PRs reviewed/commented on
gh search prs --reviewed-by={github_username} --owner=eero-inc --merged={start}..{end} --limit 100 --json number,title,url,repository
gh search prs --commenter={github_username} --owner=eero-inc --created={start}..{end} --limit 100 --json number,title,url,repository
```

#### 1B. Amazon Code Browser Activity

Fetch shipped code reviews:
```
https://code.amazon.com/reviews/from-user/{amazon_username}?shipped=true&start_time={start}%2000:00:00%20%200000&end_time={end}%2000:00:00%20%200000
```

Extract: code reviews shipped (count + details), packages contributed to.

#### 1C. JIRA Activity

Query using available JIRA tools (MCP JQL search or API with token from .zshrc):

```
# Issues assigned and worked
assignee = "{jira_name}" AND updated >= "{start}" AND updated <= "{end}" ORDER BY priority ASC, updated DESC

# Issues resolved
assignee = "{jira_name}" AND status IN ("Done", "Closed", "Resolved") AND updated >= "{start}" AND updated <= "{end}"

# Issues reported/filed
reporter = "{jira_name}" AND created >= "{start}" AND created <= "{end}" ORDER BY priority ASC, created DESC
```

Extract: issues assigned, resolved, reported, severity breakdown, issue types.

#### 1D. TestRail Activity

Use TestRail API (base URL: `https://testrail.eero.amazon.dev`, credentials from environment or .zshrc `TESTRAIL_API_KEY`).

**Test Cases Authored/Updated:**
```
GET /api/v2/get_cases/{project_id}&suite_id={suite_id}&updated_after={start_timestamp}&updated_by={user_id}
```
- Iterate all suites in project
- Filter cases where `updated_on` falls within period
- Count and categorize by suite/feature area

**Test Executions:**
```
GET /api/v2/get_runs/{project_id}&created_after={start_timestamp}&created_before={end_timestamp}
GET /api/v2/get_tests/{run_id}
GET /api/v2/get_results/{test_id}
```
- For each run in period, check results where `created_by` matches user
- Count total executions, unique test cases executed, breakdown by project/suite

**User ID resolution:**
```
GET /api/v2/get_user_by_email&email={testrail_email}
```

### Phase 2: Interactive Curation

Present the gathered data as a structured summary to the user:

```markdown
## Activity Data Collected

### GitHub
- X PRs authored across Y repos
- Z PRs reviewed
- [list top PRs by significance]

### Amazon Code Browser
- N code reviews shipped
- [list packages]

### JIRA
- A issues assigned, B resolved, C reported
- Severity: P0(x), P1(y), P2(z), P3(w)

### TestRail
- N test cases authored/updated
- M test executions across K projects
- Breakdown by feature area

### Projects Without Code Context
[Display any projects_context provided]
```

Then ask:

> Here is the raw activity data I gathered. Before I draft your work summary:
> 1. Are there any contributions or projects missing that you want included? (especially non-code work: process improvements, mentoring, cross-team initiatives, tooling)
> 2. Which contributions do you want me to emphasize as your top 3-4 highlights?
> 3. Any specific quantified outcomes you want me to call out? (e.g., "reduced regression time by X%", "enabled Y feature to ship on schedule")

### Phase 3: Draft Work Summary

Generate the work summary using the official template structure. Align contributions to:
- **QAE Role Guideline dimensions** (scope, execution, impact, technical, process improvement)
- **Amazon Leadership Principles** where naturally applicable
- **Customer impact** and business outcomes

#### Output Template

```markdown
# 2026 Work Summary: {Name}, {Role}

### 2026 H1 - Work Summary

1. How did you help our customers in the last 6 months? How did this drive business impact?

**{Contribution 1 Title} ({Month Range})** - {description of the contribution, why it matters to customers, engineering complexity, end result with quantified outcome where possible} [Leadership Principles]

**{Contribution 2 Title} ({Month Range})** - ...

**{Contribution 3 Title} ({Month Range})** - ...

[Additional contributions as warranted]

### 2. How are you helping yourself and our wider team deliver more for customers using AI?

{Description of AI usage in daily workflow, teaching others, measurable impact, limitations found}

### 3. Please reflect on any learnings you have had. How can you apply them in the next 6 months to help our customers?

{Key learnings, areas for growth, forward-looking goals}

### 4. How did you help grow or elevate those around you?

{Mentoring, code reviews, knowledge sharing, onboarding help — with specific examples and links where possible}
```

#### Writing Guidelines

1. **< 2 pages total** — concise, impactful, no filler
2. **Quantify everything possible** — test cases written, executions, bugs found, PRs reviewed, time savings
3. **Customer-first framing** — every contribution connects back to customer impact or product quality
4. **Complexity callouts** — mention constraints, ambiguity, cross-team coordination
5. **Show, don't tell** — link to specific PRs, CRs, JIRA tickets, TestRail results
6. **Leadership Principles** — tag naturally (Ownership, Deliver Results, Dive Deep, Earn Trust, etc.) without forcing
7. **Role guideline alignment** — use language that maps to QAE II expectations (difficult scenarios, reusable solutions, process improvement, mentoring)

### Phase 4: Review and Refine

Present the draft and ask:
> Here is your work summary draft. Please review and let me know:
> - Anything to add, remove, or rephrase?
> - Any factual corrections needed?
> - Want me to strengthen any section?

Iterate until the user is satisfied.

### Phase 5: Final Output

Save the final document to:
```
~/Projects/qae-skills/projects/work-summary/Work_Summary_H1_2026.md
```

Offer to copy to clipboard in plain text (no markdown formatting) for pasting into Google Docs.

## QAE II Role Guideline Reference

Use this to frame contributions appropriately:

**Scope:** Tests major features; may influence test approach of related teams.

**Execution:** Uses technical expertise to create, execute, and optimize test plans and automation for difficult scenarios. Designs reusable and reliable SOPs and test solutions. Mitigates quality risks. Defines test requirements, clears blockers, and escalates appropriately.

**Impact:** Impacts QAT/QAE efficiency, product quality, and launch/time to production.

**Technical:** Proficiency in test methodologies, scripting, and problem solving. Understands test approaches; knows when/not to apply.

**Process Improvement:** Optimizes test plans, automated solutions, and QA metrics.

**Moving to QAE III indicators:**
- Leads design/implementation of test plans in ambiguous and complex scenarios
- Maximizes team efficiency and product test coverage
- Proactively improves consistency between team test processes
- Influences product quality strategy
- Documentation and test cases set an example to others
- Demonstrates quality influence over multiple teams
- Makes impactful process changes with measurable success
- Actively mentors others

## Notes

- The work summary is NOT a performance evaluation — it is a self-advocacy document
- Focus on outcomes over activities (what shipped, what improved, what was prevented)
- Include both individual contributions and force-multiplier work (reviews, mentoring, tooling)
- Reference the role guideline language naturally — do not copy/paste guideline text verbatim
- Keep AI section honest about both benefits and limitations discovered
- The learnings section should show self-awareness and growth mindset, not just list failures

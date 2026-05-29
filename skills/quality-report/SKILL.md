---
name: quality-report
description: Use when asked to generate a quality report for a feature — covering test coverage, risk assessment, automation status, and release readiness. Works with Jira, TestRail, and GitHub data sources when available.
---

# Quality Report Skill

Generate a comprehensive quality report for a feature or project, assessing test coverage, risk areas, automation status, and release readiness with per-scope QA sign-off.

## Agent

Use the `lead-quality-assurance-engineer` agent to execute this skill.

## Inputs

Before generating the report, gather ALL of the following from the user. **Do not proceed until every required input is provided.**

### Required
1. **Feature or project name** — what is being assessed
2. **Jira epic(s)** — one or more epic keys to scope the analysis
3. **Scopes to evaluate** — e.g., "Android, iOS" or "Residential, Business"
4. **Target release version/date** — e.g., "6.60" or "2026-05-15"
5. **Release stage** — is the feature already released to Production, or still in Beta/Staging only? This determines how bugs are classified (pre-production vs. escaped) and which metrics are relevant.
6. **Test plan document link** — Confluence page, Google Doc, or other test plan reference

### Optional
7. **TestRail project/suite** — project ID, suite ID, or TestRail URL for test coverage data
8. **Project tracking link** — link to Jira board, project page, or roadmap
9. **GitHub repo** — to analyze PR activity, code churn, and review coverage
10. **Jira filter for known issues** — saved filter URL showing all known issues for the feature
11. **Time period** — defaults to current sprint/release cycle

### Gate Rule

If the user provides only a feature name or epic key, **ask for the remaining required inputs before proceeding**. Present a checklist of what's missing:

```
Before I generate the report, I need the following:
- [ ] Scopes to evaluate (e.g., Android, iOS)
- [ ] Target release version or date
- [ ] Release stage (already in Production, or Beta/Staging only?)
- [ ] Test plan document link

Please provide these so I can generate a complete report.
```

Only proceed once all required inputs are confirmed.

## Step 1: Gather Feature Context

Collect information about the feature from available sources:

1. **From Jira** — if an epic key is provided, automatically build and execute queries:

   **a) Get epic metadata:**
   - Call `jira_get_issue` on the epic key to get project key, summary, description, and linked issues.

   **b) Pull all child issues (stories/tasks):**
   ```
   "Epic Link" = [EPIC-KEY] ORDER BY status ASC, priority ASC
   ```
   If no results (next-gen/Team-managed project), fall back to:
   ```
   parent = [EPIC-KEY] ORDER BY status ASC, priority ASC
   ```

   **c) Pull all bugs linked to the epic (for metrics + Known Issues):**
   ```
   issuetype in (Bug, Defect) AND "Epic Link" = [EPIC-KEY] ORDER BY priority ASC
   ```
   Fallback for next-gen projects:
   ```
   issuetype in (Bug, Defect) AND parent = [EPIC-KEY] ORDER BY priority ASC
   ```
   If the epic spans multiple projects or bugs aren't direct children:
   ```
   issuetype in (Bug, Defect) AND issue in linkedIssues([EPIC-KEY]) ORDER BY priority ASC
   ```

   **d) Pull open bugs only (for Known Issues section and QA Sign-off blockers):**
   ```
   issuetype in (Bug, Defect) AND "Epic Link" = [EPIC-KEY] AND status != Done ORDER BY priority ASC
   ```

   **e) From the results, extract:**
   - Status of each issue (Done, In Progress, To Do)
   - Priority distribution
   - Components and labels
   - Open Sev-1/Sev-2 bugs → Blocking Issues in QA Sign-off
   - All open bugs → Known Issues table
   - Total bug count → Development Status metrics

2. **From the user** — if no Jira data is available, ask for: feature scope, platforms, key components, and known risks
3. **From GitHub** (optional) — if a repo is provided, use `get_pull_request` and `get_pull_request_files` to understand code change scope

## Step 2: Collect Timeline and Resources

Gather from user input or Jira:
- Target release version and date
- Project tracking link
- Test plan document link
- TestRail suite link
- Jira filter for known issues

## Step 3: Assess Test Coverage

### With TestRail

If TestRail is available:

1. Call `get_projects` to find the project, then `get_suites` and `get_cases` to fetch test cases
2. Filter test cases linked to the feature (via `refs` field matching Jira keys)
3. Call `get_case_fields` to identify the automation type field
4. For active test runs, call `get_runs` and `get_results_for_run` to get execution status
5. Build a coverage matrix:
   - Total test cases mapped to the feature
   - Automated vs. manual breakdown
   - Execution status: passed, failed, blocked, untested
   - Priority distribution of test cases (P0/P1/P2/P3)

### Without TestRail

If TestRail is not available, assess coverage from:
- Jira ticket descriptions and comments mentioning test cases
- PR descriptions with testing notes
- Ask the user for known coverage gaps

## Step 4: Bug Metrics

Consolidate all bug data from the epic(s) into a unified metrics view. This section provides a complete picture of defect health for the feature.

### Data Collection

From the bugs already pulled in Step 1, classify and aggregate:

**a) Pre-production vs. Escaped Bugs:**
- **If the feature is Beta/Staging only**: all bugs are pre-production by definition. The "Escaped" category does not apply — report all bugs as pre-production and note that escaped bug metrics will become relevant post-launch.
- **If the feature is already in Production**:
  - **Pre-production** — bugs found before the production release (during development, QA, or staging). Identify by: created date before release date AND (labels contain "qa-found", "dev-found", or Bug Source/Reason field indicates internal discovery, OR no "production" / "customer-reported" label).
  - **Escaped** — bugs found after release or reported by customers. Identify by: labels contain "escaped", "production", "customer-reported", OR Bug Source field = "Production" / "Customer", OR created date after release date with production indicators.
- If Bug Source/Reason custom fields exist in the project, use those as the primary classifier. Otherwise fall back to labels and date heuristics.

**b) Bug Status Breakdown:**
- Open (To Do / New / Backlog)
- In Progress (In Development / In Review)
- Awaiting Verification (Ready for QA / In QA / Awaiting Verification)
- Closed (Done / Resolved / Won't Do)

Map project-specific statuses to these four buckets.

**c) Severity by Scope:**
For each scope the user provided, count bugs by priority (P0, P1, P2, P3, P4).

**d) Resolution Rate:**
- Total bugs opened during the analysis period
- Total bugs closed during the analysis period
- Resolution rate = closed / opened × 100
- Net open trend: are bugs accumulating or being resolved faster than filed?

**e) Pending Bugs Needing Action:**
- Triaged but unassigned (status != Backlog, assignee = empty)
- Assigned but stale (assignee set, no status change in >5 days)
- Awaiting verification (fix merged but not yet verified by QA)

**f) Bugs by Component:**
Group bugs by Jira component field. If components aren't used, group by label or affected area from the summary/description.

**g) Bugs by Assignee:**
Count open bugs per assignee to identify bottlenecks or unbalanced workload.

**h) Age of Open Bugs:**
For all open bugs, calculate days since creation. Bucket into:
- < 3 days
- 3–7 days
- 7–14 days
- > 14 days

Flag any P0/P1 bugs older than 7 days as stale blockers.

## Step 5: Risk Analysis

Evaluate risk across these dimensions:

| Dimension | How to Assess |
|-----------|---------------|
| **Coverage gaps** | Features/flows with no test cases or only manual coverage |
| **Code churn** | Files changed frequently in recent PRs (from GitHub) — high churn = higher regression risk |
| **Bug density** | Count of bugs filed against the feature during development (from Jira) |
| **Dependency risk** | External service dependencies, third-party integrations identified from tech spec or Jira |
| **Platform risk** | Cross-platform features with uneven test coverage |
| **Automation debt** | P0/P1 scenarios that are manual-only |

Assign each dimension a risk level: **High**, **Medium**, or **Low**.

## Step 6: Identify Post-Launch Risks

Separately from the risk matrix, identify risks that could affect users after launch:
- Known open issues that users could reproduce
- Dependencies on external actions (e.g., website pages being published, third-party services)
- Workarounds available for each risk
- Severity of user impact

## Step 7: QA Sign-off (Per-Scope)

Evaluate release readiness **per scope** (not as a single verdict). For each scope provided by the user, assess:

1. **Story completion** — % of stories in Done status for that scope
2. **Bug status** — open bugs by severity for that scope, any open Sev-1/Sev-2 blockers
3. **Test execution** — % of P0/P1 test cases executed and passing for that scope
4. **Automation coverage** — % of P0/P1 test cases automated for that scope
5. **Known risks** — unmitigated high-risk items from Steps 5–6

Produce a per-scope verdict:
- 🟢 **GO** — all criteria met, safe to release
- 🟡 **GO with Risks** — criteria mostly met, known risks accepted
- 🔴 **NO-GO** — blocking issues prevent release for this scope

If only one scope exists, produce a single verdict.

## Step 8: Conditions for Sign-off

Based on the analysis, present findings as observations — not directives. The report is QA's professional assessment shared with product and engineering peers. Use declarative statements of current state rather than imperative commands.

Structure into three tiers:
1. **Before sign-off can proceed** — what QA needs to see resolved to change the verdict (state the facts, not the actions)
2. **Outstanding items not blocking, but carrying risk** — unscheduled issues, missing coverage, under-tested areas
3. **Observations for future milestones** — patterns, stabilization needs, technical debt

## Output Format

Generate a **condensed 1-pager** report saved to `~/quality-reports/quality-report-[feature-name]-[date].md`. The report must be formatted for direct copy-paste into a Confluence page (markdown tables, horizontal rules as section dividers, no nested headers beyond h3).

```markdown
# Quality Report: [Feature Name]

**Date**: [date] | **Release**: [version] | **Epics**: [epic links]

---

## Feature Overview

[2–3 sentences: what the feature does, why it exists, what platforms/scopes are affected]

---

## QA Sign-off

| Scope | Verdict | Justification |
|-------|---------|---------------|
| [Scope 1] | 🟢/🟡/🔴 [GO/GO with Risks/NO-GO] | [1-line justification] |

### Blocking Issues

| Key | Summary | Severity | Scope | Status |
|-----|---------|----------|-------|--------|
| [JIRA-KEY] | [summary] | [P0/P1] | [scope] | [status] |

---

## Development Status

| Metric | [Scope 1] | [Scope 2] |
|--------|----------:|----------:|
| Done | [n] ([%]) | [n] ([%]) |
| In Progress | [n] ([%]) | [n] ([%]) |
| To Do | [n] ([%]) | [n] ([%]) |
| Won't Do | [n] ([%]) | [n] ([%]) |
| Open P0/P1 Bugs | [n] | [n] |

---

## Bug Metrics

### Pre-production vs. Escaped

| Category | Count | % of Total |
|----------|------:|----------:|
| Pre-production (found before release) | [n] | [%] |
| Escaped (found post-release / customer-reported) | [n] | [%] |
| **Total** | [n] | 100% |

### Status Breakdown

| Status | Count |
|--------|------:|
| Open | [n] |
| In Progress | [n] |
| Awaiting Verification | [n] |
| Closed | [n] |

### Severity by Scope

| Severity | [Scope 1] | [Scope 2] |
|----------|----------:|----------:|
| P0 | [n] | [n] |
| P1 | [n] | [n] |
| P2 | [n] | [n] |
| P3+ | [n] | [n] |

### Resolution Rate

| Metric | Value |
|--------|------:|
| Bugs opened (period) | [n] |
| Bugs closed (period) | [n] |
| Resolution rate | [%] |
| Net trend | [accumulating / resolving] |

### Pending Bugs Needing Action

| Category | Count | Details |
|----------|------:|---------|
| Triaged but unassigned | [n] | [keys] |
| Assigned but stale (>5 days) | [n] | [keys] |
| Awaiting verification | [n] | [keys] |

### Bugs by Component

| Component | Open | Closed | Total |
|-----------|-----:|-------:|------:|
| [component] | [n] | [n] | [n] |

### Bugs by Assignee

| Assignee | Open Bugs |
|----------|----------:|
| [name] | [n] |

### Age of Open Bugs

| Age Bucket | Count | P0/P1 Stale? |
|-----------|------:|:------------:|
| < 3 days | [n] | — |
| 3–7 days | [n] | — |
| 7–14 days | [n] | ⚠️ [keys] |
| > 14 days | [n] | 🔴 [keys] |

---

## Risks

| Risk | Severity | Impact |
|------|----------|--------|
| [description] | [High/Med/Low] | [user impact] |

---

## QA Resources

| Resource | Link |
|----------|------|
| Test Plan | [link] |
| TestRail Suite | [link or Pending] |
| Jira | [epic links] |

---

## Conditions for Sign-off

**Before sign-off can proceed:**
- [Declarative statement of what remains unresolved, with issue references]

**Outstanding items not blocking, but carrying risk into release:**
- [Observations about unscheduled issues, missing coverage, under-tested areas]

**Observations for future milestones:**
- [Patterns noticed, stabilization needs, technical debt]
```

### Report Naming Convention

- Feature report: `quality-report-[feature-name]-[YYYY-MM-DD].md`
- Sanitize feature name: lowercase, hyphens for spaces, strip special characters

## Critical Rule: Always Save Report to File

Every execution of this skill MUST write the full report to a markdown file in `~/quality-reports/` before presenting results in chat. After saving, confirm the file path to the user.

## Workflow Summary

1. User requests a quality report for a feature
2. Skill gathers feature context from Jira, GitHub, and/or user input
3. Skill collects timeline and resource links
4. Skill assesses test coverage from TestRail and/or available data
5. Skill consolidates bug metrics (pre-production vs escaped, status, severity, resolution rate, pending, by component, by assignee, age)
6. Skill evaluates risk across six dimensions
7. Skill identifies post-launch risks (user-facing)
8. Skill determines QA sign-off verdict per scope
9. Skill generates actionable recommendations
10. **Skill writes the full report to `~/quality-reports/quality-report-[feature]-[date].md`**
11. Skill confirms the saved file path to the user
12. Skill presents the executive summary and QA sign-off in chat

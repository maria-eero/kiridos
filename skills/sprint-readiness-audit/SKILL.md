---
name: sprint-readiness-audit
description: Use when asked to audit a newly created sprint for gaps that affect SDLC flow. Checks for missing story points, empty QA assignment, empty bug reason/source, and missing descriptions. Produces a concise, actionable report for the SDM to address critical gaps before sprint execution begins.
---

# Sprint Readiness Audit

Audit a sprint at kickoff to surface critical gaps — missing story points, empty QA assignments, unfilled bug metadata, and tickets lacking descriptions — then produce a concise report the SDM can act on immediately.

## Agents

This skill uses a **three-agent pipeline**:

1. `program-manager-iii` — assesses overall sprint readiness from a program delivery perspective: are dependencies mapped, is scope clear, are success criteria defined? Flags systemic process gaps.
2. `lead-quality-assurance-engineer` — audits quality-related fields: QA assignment, bug reason, bug source, and test coverage signals. Flags tickets that will create downstream QA bottlenecks.
3. `software-developer-manager-iii` — synthesizes findings into a prioritized action plan with ownership assignments and sprint health risk assessment.

## Prerequisites

- **Jira MCP Server**: `mcp-atlassian` configured with access to the target project.

## Workflow

### Step 1: Gather Sprint ID

Ask the user:

> What is the Sprint ID to audit? (You can find this in the Jira board sprint URL)

**Do not proceed until the Sprint ID is provided.** There is no default — the user must specify one.

### Step 2: Fetch Sprint Tickets

Use `jira_get_sprint_issues` or `jira_search` with:

```
sprint = [SPRINT_ID] ORDER BY issuetype ASC, priority ASC
```

Request fields: `key`, `summary`, `issuetype`, `status`, `priority`, `assignee`, `customfield_10005` (Story Points), `customfield_11787` (QA Assignee), `description`, `components`, `labels`, `customfield_11560` (Bug Reason), `customfield_11594` (Bug Source), `customfield_11685` (Bug Source - Category).

**Story Points field**: Use `customfield_10005`. A value of `0.0` is a valid estimate (not missing). Only `null` counts as missing.

**QA Assignee field**: Use `customfield_11787`. This is a user picker field — `null` means no QA is assigned.

Paginate until all tickets are retrieved.

### Step 3: Run Gap Detection

For each ticket, check the following gaps:

| Gap | Condition | Severity |
|-----|-----------|----------|
| **Missing Story Points** | Story/Task/Bug with no story points | 🔴 Critical — blocks capacity planning |
| **Empty QA Assignment** | Ticket has no QA assignee field populated | 🟡 High — risks QA bottleneck |
| **Empty Bug Reason** | Bug/Defect with no Bug Reason field | 🟡 High — blocks root cause tracking |
| **Empty Bug Source** | Bug/Defect with no Bug Source field | 🟡 High — blocks escape analysis |
| **Missing Description** | Any ticket with empty or <20 char description | 🔴 Critical — requirements unclear |
| **No Components** | Ticket has no component assigned | 🟠 Medium — affects routing and reporting |
| **Unassigned Ticket** | No assignee at all | 🟠 Medium — ownership unclear |

### Step 4: Program Manager Assessment

The `program-manager-iii` agent reviews the gap data and produces:

- **Scope Clarity Score**: % of tickets with sufficient description + story points
- **Delivery Risk**: Are there tickets with no owner, no points, AND no description? These are "blind spots" — work that will stall or surprise the team mid-sprint.
- **Process Gap Pattern**: Are the same gap types recurring? (e.g., "80% of bugs lack Bug Reason — this is a filing discipline issue, not a one-off")

### Step 5: QA Readiness Assessment

The `lead-quality-assurance-engineer` agent reviews and produces:

- **QA Coverage Risk**: % of tickets missing QA assignment — these won't get tested unless someone picks them up
- **Bug Metadata Health**: % of bugs missing Bug Reason and/or Bug Source — these gaps break downstream reporting (escape analysis, defect management reports)
- **Testability Flags**: Tickets with no description are untestable — flag them explicitly

### Step 6: SDM Action Plan

The `software-developer-manager-iii` synthesizes both assessments into:

- **Sprint Health Score**: 🔴/🟡/🟢 based on gap severity and volume
- **Top 5 Actions**: Ordered by impact, each with a suggested owner
- **Capacity Impact**: Estimate of how many unpointed tickets affect sprint commitment accuracy

### Step 7: Generate Report

Compose the report in this format:

```markdown
# Sprint Readiness Audit — [Sprint Name]

**Project**: [name] | **Sprint**: [name] | **Total Tickets**: [n] | **Audit Date**: [today]

---

## 🏥 Sprint Health: [🔴/🟡/🟢]

[1-sentence summary: e.g., "34% of tickets have critical gaps that will disrupt delivery if not addressed before sprint execution begins."]

---

## Gap Summary

| Gap Type | Count | % of Sprint | Severity |
|----------|------:|------------:|----------|
| Missing Story Points | [n] | [%] | 🔴 Critical |
| Missing Description | [n] | [%] | 🔴 Critical |
| Empty QA Assignment | [n] | [%] | 🟡 High |
| Empty Bug Reason | [n] | [%] | 🟡 High |
| Empty Bug Source | [n] | [%] | 🟡 High |
| No Components | [n] | [%] | 🟠 Medium |
| Unassigned | [n] | [%] | 🟠 Medium |

**Scope Clarity Score**: [X]% of tickets are fully specified (description + points + assignee)

---

## 🔴 Critical Gaps — Immediate Action Required

### Missing Story Points ([n] tickets)

| Key | Summary | Type | Assignee |
|-----|---------|------|----------|
| [KEY](url) | [summary] | Story | [name] |

### Missing Description ([n] tickets)

| Key | Summary | Type | Assignee |
|-----|---------|------|----------|
| [KEY](url) | [summary] | Bug | [name] |

---

## 🟡 High Gaps — Address Within 24h

### Empty QA Assignment ([n] tickets)

| Key | Summary | Type | Assignee |
|-----|---------|------|----------|
| [KEY](url) | [summary] | Story | [name] |

### Empty Bug Metadata ([n] tickets)

| Key | Summary | Missing Fields |
|-----|---------|----------------|
| [KEY](url) | [summary] | Bug Reason, Bug Source |

---

## 🟠 Medium Gaps

[Brief table or count — no detailed listing unless <5 tickets]

---

## Decisions & Actions

> Prioritized for the SDM to assign before sprint execution begins.

1. **[Action]** — Owner: [suggested] — Impact: [what it unblocks]
2. **[Action]** — Owner: [suggested] — Impact: [what it unblocks]
3. **[Action]** — Owner: [suggested] — Impact: [what it unblocks]
4. **[Action]** — Owner: [suggested] — Impact: [what it unblocks]
5. **[Action]** — Owner: [suggested] — Impact: [what it unblocks]

**Capacity Note**: [n] unpointed tickets represent unknown effort. Sprint commitment accuracy is [low/moderate/high].

---

*Generated by Sprint Readiness Audit | [date]*
```

### Step 8: Deliver Report

1. **Display** the report directly to the user in chat.
2. **Ask** if they want it:
   - Saved locally to `~/sprint-readiness-reports/`
   - Published to Confluence (ask for space and parent page)
   - Both

## Design Principles

- **Concise over comprehensive** — the SDM needs to act in 5 minutes, not read for 20.
- **Actionable over analytical** — every gap listed must have a clear "fix this" path.
- **Severity-ordered** — critical gaps first, medium gaps summarized.
- **No opinions on ticket content** — this audit checks field completeness, not whether the description is *good enough*. That's a separate review.

## Thresholds for Health Score

- 🔴 **Red**: >25% of tickets have critical gaps (missing points OR missing description)
- 🟡 **Yellow**: 10–25% critical gaps OR >40% high gaps (QA/bug metadata)
- 🟢 **Green**: <10% critical gaps AND <25% high gaps

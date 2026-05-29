---
name: sprint-bug-health-report
description: Use when asked to generate a sprint bug health report for Core or SWS projects. Fetches bugs/defects from a sprint, generates visual-friendly tables and metrics, and publishes a concise report to Confluence focused on actionable insights for upcoming sprints.
---

# Sprint Bug Health Report

Generate a focused sprint bug health report with metrics, trend indicators, and actionable recommendations to help the team reduce bug volume in upcoming sprints. Published directly to Confluence.

## Agents

Use **both** of the following agents collaboratively:

- `lead-quality-assurance-engineer` — drives the technical analysis: bug categorization, severity distribution, root cause patterns, and test coverage gaps.
- `software-developer-manager-iii` — drives the strategic lens: team impact assessment, resource allocation insights, sprint-over-sprint trends, and prioritization recommendations for upcoming sprints.

The QAE agent produces the data and analysis sections. The SDM agent reviews the analysis and produces the "Decisions & Recommendations" section with a leadership perspective on what to prioritize next.

## Prerequisites

- **Jira MCP Server**: `mcp-atlassian` configured with access to the `Core Engineering` and `SWS` projects.
- **Confluence MCP Server**: `mcp-atlassian` configured with write access to publish the report.

## Workflow

### Step 1: Prompt for Project and Sprint

Ask the user:

> Which project would you like to generate the sprint bug health report for?
> 1. **Core Engineering** (Core)
> 2. **Software Services** (SWS)

If the user selects **Core Engineering**, follow up with:

> Which Core Engineering team?
> 1. **Onboarding & Insight** (OnbrdInsight)
> 2. **Core Network Management** (CNM)

Then ask:

> And which sprint ID should I pull bugs from? (You can find this in the Jira board sprint URL)
> Also, what is the previous sprint's ID and name? (needed for trend comparison)

**Do not proceed until the project, team (if Core), sprint ID, and previous sprint ID are all provided.** There is no default sprint — the user must specify one for the report to be accurate. The previous sprint is required for trend columns in every section.

**Sprint name patterns by team:**
- Onboarding and Insight: `CoreOnbdInsight - Sprint [version]` (board 4155)
- Core Network Management: `CNM - Sprint [version]` (use sprint ID to discover)

**File naming convention:**
- OnbrdInsight: `defect-management-report-OnbrdInsight-[sprint-id]-[date].md`
- CNM: `defect-management-report-CNM-[sprint-id]-[date].md`
- SWS: `defect-management-report-SWS-[sprint-id]-[date].md`

### Step 2: Fetch Sprint Bugs from Jira (Current + Previous)

Based on the user's project selection, execute the appropriate JQL query using `jira_search`:

**For Core Engineering:**
```
project = "Core Engineering" AND type IN (Bug, Defect) AND sprint = [SPRINT_ID] ORDER BY created DESC
```

**For SWS:**
```
project = "Software Services (SWS)" AND type IN (Bug, Defect) AND sprint = [SPRINT_ID] ORDER BY created DESC
```

Request these fields: `key`, `summary`, `priority`, `status`, `assignee`, `components`, `labels`, `created`, `resolved`, `resolution`, `customfield_11560` (Bug Reason), `customfield_11594` (Bug Source), `customfield_11685` (Bug Source - Category), `customfield_10900` (Severity), `platform`, `parent` (Epic Link).

Paginate until all results are retrieved.

**IMPORTANT — Also fetch the previous sprint's bugs using the same query and fields.** The user must provide the previous sprint ID (or it can be inferred from the sprint name pattern, e.g., "26.5" for current "26.6"). This data is required to populate trend columns in every section of the report. If the previous sprint ID is not available, mark trend columns as "—" and note "baseline established" for future comparison.

### Step 3: Analyze Bug Data (QAE Agent)

The `lead-quality-assurance-engineer` produces the following analysis:

**a) Severity Distribution**

Show both Priority (fix urgency) and Severity (customer impact). **All tables must include previous sprint data in a side-by-side comparison format with a Trend column.**

**By Priority (Fix Urgency):**

| Priority | Sprint [current] | Sprint [previous] | Trend |
|----------|------:|----------:|-------|
| P0 (Critical) | count (%) | count (%) | ↑/↓/→ +/-n |
| P1 (High) | count (%) | count (%) | ↑/↓/→ +/-n |
| P2 (Medium) | count (%) | count (%) | ↑/↓/→ |
| P3 (Low) | count (%) | count (%) | ↑/↓/→ |

Add an inline commentary note below the table (e.g., *"⚠️ P0/P1 concentration rose from 42% to 52% — driven by [reason]."*)

**By Severity (Customer Impact)** — using `customfield_10900`. Only include bugs where Severity is populated. Note the fill rate.

| Severity | Count | % | Meaning |
|----------|------:|--:|---------|
| 1 - Critical | | | Complete loss of functionality, no workaround |
| 2 - Major | | | Major feature broken, workaround exists |
| 3 - Moderate | | | Feature partially impacted |
| 4 - Minor | | | Cosmetic or low-impact issue |

**b) Status Breakdown**

| Status | Count | % |
|--------|------:|--:|
| Open | | |
| In Progress | | |
| Done/Closed | | |

**c) Component Hotspots** — top 5 components by bug count, with previous sprint comparison:

| Component | Sprint [current] | Sprint [previous] | Trend | Action Needed |
|-----------|-----:|-----:|-------|---------------|
| [name] | [n] | [n] | ↑/↓/→ +/-n | [brief action] |

**d) Platform Hotspots** — group bugs by the `platform` field (e.g., iOS, Android, eeroOS, Backend, Web) with previous sprint comparison:

| Platform | Sprint [current] | Sprint [previous] | Trend | % of Total |
|----------|-----:|-----:|-------|----------:|
| [platform] | [n] | [n] | ↑/↓/→ +/-n | [%] |

Note the fill rate (% of bugs with platform populated). If platform is sparsely populated, flag as a process gap in recommendations.

**e) Bugs by Epic** — aggregate bugs by their parent epic to show which feature areas are generating the most defects:

| Epic | Key | Sprint [current] | Sprint [previous] | Trend | P0/P1 |
|------|-----|-----:|-----:|-------|------:|
| [epic summary] | [EPIC-KEY] | [n] | [n] | ↑/↓/→ | [n] |
| *(No epic linked)* | — | [n] | [n] | — | [n] |

Order by current sprint count descending. Include a row for bugs with no epic linked. This helps the team understand which feature scopes are driving defect volume.

**f) Bug Filing Rate** — bugs filed per week within the sprint (to show if bugs cluster early or late).

**g) Resolution Rate** — % of sprint bugs resolved within the sprint vs. carried over. **Show side-by-side with previous sprint:**

| Metric | Sprint [current] | Sprint [previous] |
|--------|------:|------:|
| Bugs resolved | n (%) | n (%) |
| Bugs carried over | n (%) | n (%) |
| Resolved via Code Changed | n | n |
| Resolved as Cannot Reproduce / Won't Do | n | n |

**h) Created vs. Resolved** — compare how many bugs were filed during the sprint vs. how many were resolved. Shows whether the team is keeping pace or accumulating debt. Use "Net reduction: X" when resolved > created (positive signal), or "<font color="red">**Net increase: X**</font>" when created > resolved (debt accumulating).

**i) Bug Age Distribution** — group open bugs by age brackets (0–7 days, 8–14 days, 15–30 days, 30+ days) to surface stale issues piling up.

**j) Sprint-over-Sprint Trend** — fetch the previous sprint's bug data (same board, use sprint name pattern to find the prior sprint, e.g., "CoreOnbdInsight - Sprint 26.5" for current "26.6"). Compare:
- Total bugs, P0/P1 count and %, resolution rate, carry-over count, customer escapes, top bug reason, top component
- Show delta and directional arrow (↑ worse / ↓ better / → stable)
- Include a **Key Highlights** subsection with 3–5 bullet points summarizing the most important changes
- If the previous sprint data cannot be determined automatically, note the comparison as "—" and mark as "baseline established."

**CRITICAL: Sprint-over-sprint comparison is NOT limited to the dedicated Trend section.** Every data table in the report (Severity, Components, Bug Source, Bug Reason, Resolution Metrics) MUST include the previous sprint's values as a column, with a Trend column showing directional arrows. This is the primary format — the dedicated Sprint-over-Sprint Trend section is a summary rollup, not the only place trends appear.

**Sprint Health section** must also include an inline comparison note below the summary (e.g., *"vs. Sprint 26.5: P0/P1 was 42% (↑ 10pp worse), resolution rate was 90%"*).

**j-2) Operational Excellence Highlights** — fetch all tickets in the sprint with the `core-operational-excellence-day` label that are resolved/done:

```
sprint = [SPRINT_ID] AND labels = "core-operational-excellence-day" AND status = Done ORDER BY assignee ASC
```

Group results by the Jira Team field (`customfield_11000`). Rank teams by count of resolved tickets. The top team gets a shoutout callout in the report. Include ticket keys as hyperlinks.

**k) Bug Source & Classification** — break down bugs by source and reason, **always with previous sprint comparison columns:**

- **Bug Source (Who Found It)**: group by the Bug Source field values with count and percentage, side-by-side with previous sprint:

| Source | Sprint [current] | Sprint [previous] | Trend |
|--------|------:|------:|-------|
| Manual Testing | n (%) | n (%) | ↑/↓/→ |
| Customer Feedback | n (%) | n (%) | ↑/↓/→ |
| Engineering Team | n (%) | n (%) | ↑/↓/→ |

- **Bug Reason (Root Cause)** (when populated): group by Bug Reason field values with previous sprint comparison:

| Bug Reason | Sprint [current] | Sprint [previous] | Trend |
|------------|------:|------:|-------|
| Incorrect Logic | n (%) | n (%) | ↑/↓/→ |

Note the fill rate (% of bugs with this field populated) — low fill rate signals a process gap to flag in recommendations.

- **Incorrect Logic Sub-classification**: When "Incorrect Logic" is the top bug reason (or represents >25% of bugs), perform a deeper analysis to sub-classify into:

| Sub-category | Count | % of Incorrect Logic | Example |
|-------------|------:|--------------------:|---------|
| Conditional/Branching Error | [n] | [%] | Wrong if/else path, missing condition |
| State Management | [n] | [%] | Stale state, race condition, incorrect state transition |
| Data Handling | [n] | [%] | Wrong transformation, incorrect parsing, type mismatch |
| Boundary/Edge Case | [n] | [%] | Off-by-one, null handling, empty collection |
| Integration Contract | [n] | [%] | Wrong API usage, incorrect payload, mismatched interface |
| Business Rule Violation | [n] | [%] | Requirement misinterpretation, wrong calculation |
| Concurrency/Timing | [n] | [%] | Race condition, deadlock, incorrect async handling |
| Other | [n] | [%] | Doesn't fit above categories |

**Classification method — GitHub PR analysis (primary):**

For each bug with "Incorrect Logic" as the bug reason:
1. Check the Jira issue for a linked GitHub PR (use `jira_get_issue_development_info` to find linked pull requests).
2. If a PR is linked, use `get_pull_request` and `get_pull_request_files` to read the fix PR's diff and description.
3. Classify based on what the code change actually fixed:
   - Changed conditional logic, added/removed if-else branches → **Conditional/Branching Error**
   - Fixed state variables, lifecycle hooks, store/reducer logic → **State Management**
   - Fixed parsing, serialization, type conversion, data mapping → **Data Handling**
   - Added null checks, boundary guards, empty-state handling → **Boundary/Edge Case**
   - Fixed API call parameters, response handling, contract alignment → **Integration Contract**
   - Corrected a calculation, formula, or business rule implementation → **Business Rule Violation**
   - Fixed async/await, threading, timing, or race condition → **Concurrency/Timing**
4. If no PR is linked or the PR diff is inconclusive, fall back to reading the bug summary + description for keyword signals.
5. If still ambiguous, bucket as "Other" and note the ambiguity.

This sub-classification helps engineering target specific code review practices or testing strategies (e.g., if "State Management" dominates, recommend state machine testing).

- **Resolution Breakdown** (resolved bugs only): group by Resolution field (Code Changed, Cannot Reproduce, Won't Do, Duplicate, etc.) to show how bugs are being closed.

### Step 4: Strategic Assessment (SDM Agent)

The `software-developer-manager-iii` reviews the QAE analysis and produces:

**a) Sprint Health Score** — a simple Red/Yellow/Green indicator based on:
- 🔴 Red: >30% of bugs are P0/P1 OR resolution rate <50%
- 🟡 Yellow: 15–30% P0/P1 OR resolution rate 50–75%
- 🟢 Green: <15% P0/P1 AND resolution rate >75%

**b) Top 3 Recommendations** — actionable items for the next sprint to reduce bug volume. Each recommendation should be:
- Specific (name the component, area, or practice)
- Measurable (what would success look like)
- Achievable within 1–2 sprints

**c) Carry-over Risk** — assessment of unresolved bugs and their impact on next sprint capacity.

### Step 5: Generate the Report

Compose the report using this structure:

```markdown
# Defect Management Report — [Project Name]

**Sprint**: [Sprint Name] | **Dates**: [start] – [end], [year] | **Total Bugs**: [count] ([carry-over] carry-over + [new] filed this sprint)

---

## Table of Contents

(Use Confluence TOC macro — excludes Sprint Health)

---

## 🏥 Sprint Health: [🔴/🟡/🟢] [Red/Yellow/Green]

[1–2 sentence summary of overall health and the primary concern or positive signal]

| Metric | Value |
|--------|------:|
| Total Bugs | [n] |
| Carry-over (from previous sprint) | [n] ([%]) |
| Filed this sprint | [n] ([%]) |
| Resolved this sprint | [n] ([%]) |
| Still open | [n] |

*vs. Sprint [previous]: P0/P1 was X% (↑/↓ Npp), resolution rate was Y% (↑/↓ Npp).*

*Carry-over determined by comparing ticket `created` date against sprint start date. Tickets created before the sprint start date are carry-overs.*

---

## Severity Distribution

*This section covers all bugs in the sprint (both resolved and open). Of these: [n] are resolved, [n] remain open.*

### By Priority (Fix Urgency)

| Priority | Sprint [current] | Sprint [previous] | Trend | Resolved | Open |
|----------|------:|----------:|-------|------:|------:|
| **P0 (Blocker)** | **n (%)** | n (%) | ↑/↓/→ +/-n | n | **n** |
| **P1 (Critical)** | **n (%)** | n (%) | ↑/↓/→ +/-n | n | **n** |
| P2 (Major) | n (%) | n (%) | → | n | n |
| P3 (Minor) | n (%) | n (%) | → | n | n |

*[Inline commentary on trend, e.g., "⚠️ P0 bugs doubled sprint-over-sprint..."]*

### By Severity (Customer Impact)

| Severity | Count | % | Meaning |
|----------|------:|--:|---------|
| 1 - Critical | | | Complete loss of functionality |
| 2 - Major | | | Major feature broken, workaround exists |
| 3 - Moderate | | | Feature partially impacted |
| 4 - Minor | | | Cosmetic or low-impact issue |

---

## Component Hotspots

| Component | Sprint [current] | Sprint [previous] | Trend | Action Needed |
|-----------|-----:|-----:|-------|---------------|
| **[name]** | **[n]** | [n] | ↑/↓/→ | [brief action] |

*[Inline commentary on emerging/declining hotspots]*

---

## Platform Hotspots

| Platform | Sprint [current] | Sprint [previous] | Trend | % of Total |
|----------|-----:|-----:|-------|----------:|
| [platform] | [n] | [n] | ↑/↓/→ +/-n | [%] |

*[Inline commentary on platform-specific patterns, e.g., "iOS bugs rose 40% — driven by [feature area]"]*

---

## Bugs by Epic

| Key | Epic | Sprint [current] | Sprint [previous] | Trend | P0/P1 |
|-----|------|-----:|-----:|-------|------:|
| [EPIC-KEY](https://eeroinc.atlassian.net/browse/EPIC-KEY) | [epic summary] | [n] | [n] | ↑/↓/→ | [n] |
| — | *(No epic linked)* | [n] | [n] | — | [n] |

*[Commentary on which feature scopes are driving defect volume]*

---

## Bug Source & Classification

### Bug Source (Who Found It)

| Source | Sprint [current] | Sprint [previous] | Trend |
|--------|------:|------:|-------|
| **Manual Testing** | **n (%)** | n (%) | ↑/↓/→ |
| Customer Feedback | n (%) | n (%) | ↑/↓/→ |
| Engineering Team | n (%) | n (%) | → |

### Bug Reason (Root Cause)

| Bug Reason | Sprint [current] | Sprint [previous] | Trend |
|------------|------:|------:|-------|
| **Incorrect Logic** | **n (%)** | n (%) | ↑/↓/→ |

### Incorrect Logic Breakdown

*(Included when Incorrect Logic is top reason or >25% of bugs)*

| Sub-category | Count | % of Incorrect Logic |
|-------------|------:|--------------------:|
| Conditional/Branching Error | [n] | [%] |
| State Management | [n] | [%] |
| Data Handling | [n] | [%] |
| Boundary/Edge Case | [n] | [%] |
| Integration Contract | [n] | [%] |
| Business Rule Violation | [n] | [%] |
| Concurrency/Timing | [n] | [%] |
| Other | [n] | [%] |

*[Commentary on dominant sub-category and recommended engineering action]*

---

## Resolution Metrics

| Metric | Sprint [current] | Sprint [previous] |
|--------|------:|------:|
| Bugs resolved | [n (%)](JQL: sprint=ID AND status=Done AND issuetype in (Bug,Defect)) | n (%) |
| **Bugs carried over** | [**n (%)**](JQL: sprint=ID AND status!=Done AND issuetype in (Bug,Defect)) | n (%) |
| Resolved via Code Changed | [n](JQL: sprint=ID AND resolution="Code Changed" AND issuetype in (Bug,Defect)) | n |
| Resolved as Cannot Reproduce / Won't Do | [n](JQL: sprint=ID AND resolution in ("Won't Do","Cannot Reproduce") AND issuetype in (Bug,Defect)) | n |

*Each count in the current sprint column must be a JQL hyperlink filtering to exactly those issues.*

---

## Created vs. Resolved

| Metric | Count |
|--------|------:|
| Bugs created during sprint | [n] |
| Bugs resolved during sprint | [n] |
| Net change | [+/-n] |

*[Commentary: "Net reduction: N — team is clearing backlog" or "Net increase: N — debt accumulating"]*

---

## 🏆 Operational Excellence Highlights

Tickets resolved this sprint that carry the `core-operational-excellence-day` label, grouped by team. These represent proactive quality improvements presented at the Operational Excellence meeting.

| Team | Resolved | Tickets |
|------|------:|---------|
| **[Report's team name] 🥇** | **[n]** | [KEY-1](link), [KEY-2](link) |
| [Team 2] | [n] | [KEY-3](link) |
| [Team 3] | [n] | [KEY-4](link) |

> 🎉 **Shoutout to [Top Team]** for leading operational excellence this sprint with [n] resolved tickets!

*Always list ALL teams that resolved op-ex tickets. Bold/highlight the team this report is being generated for. If only one team has op-ex tickets, note that other teams had none this sprint.*

*To fetch: `jql = sprint = [SPRINT_ID] AND labels in ("core-operational-excellence-day", "Core-operational-excellence-day") AND status = Done ORDER BY assignee ASC`. Group results by the Jira Team field (`customfield_11000`).*

---

## Bug Age Distribution (Open Bugs)

| Age Bracket | Count | % of Open |
|-------------|------:|----------:|
| 0–7 days | | |
| 8–14 days | | |
| 15–30 days | | |
| **30+ days** | | |

---

## Sprint-over-Sprint Trend

| Metric | Sprint [previous] | Sprint [current] | Delta | Trend |
|--------|------:|------:|------:|-------|
| Total bugs | [n] | [n] | [+/-n] | ↑/↓/→ |
| P0/P1 % | [%] | [%] | [+/-pp] | ↑/↓/→ |
| P0 count | [n] | [n] | [+/-n] | ↑/↓/→ |
| Resolution rate | [%] | [%] | [+/-pp] | ↑/↓/→ |
| Carry-over | [n] | [n] | [+/-n] | ↑/↓/→ |
| Customer escapes | [n] | [n] | [+/-n] | ↑/↓/→ |
| Top reason | [reason] (%) | [reason] (%) | [+/-pp] | ↑/↓/→ |

### Key Highlights

- [3–5 bullet points summarizing the most important sprint-over-sprint changes]
- Use **bold** for worsening metrics, regular text for improvements
- Always include at least one "Positive:" bullet

---

## Decisions & Recommendations

> These recommendations are prioritized for the upcoming sprint to reduce bug volume.

1. **[Recommendation 1]** — [specific, measurable action]
2. **[Recommendation 2]** — [specific, measurable action]
3. **[Recommendation 3]** — [specific, measurable action]

**Carry-over impact**: [1 sentence on how unresolved bugs affect next sprint planning]

---

## Open Bugs Requiring Attention

| Key | Summary | Priority | Severity | Assignee | Days Open |
|-----|---------|----------|----------|----------|-----------|
| [key](https://eeroinc.atlassian.net/browse/key) | [summary] | **P0** | [1-Critical/2-Major/—] | [name] | [n] |

*Ordered oldest to newest. Only P0/P1 open bugs listed. Full list available in Jira.*

---

## Appendix

**JQL Used**: `[exact query]`
**Previous Sprint**: [name] (ID: [id])
**Sprint**: [name] (ID: [id])
**Generated by**: Defect Management Report Skill | **Date**: [today]
```

### Step 6: Generate Visual Dashboard

Generate a visual summary PNG using the dashboard generator at `~/sprint-bug-reports/dashboard-generator/generate.js`:

1. Create a JSON data file with the report metrics (see `data-core-14961.json` as template)
2. Run: `node ~/sprint-bug-reports/dashboard-generator/generate.js <data-file.json>`
3. The script outputs the PNG path

The dashboard includes: health badge, 4 KPI boxes, component hotspot bars, severity legend, bug source breakdown, top bug reasons, and recommendations.

### Step 7: Get Sprint Name and Date Range

Before publishing, call `jira_search` or use the sprint metadata to retrieve:
- The **sprint name** as it appears in Jira (e.g., "Sprint 42")
- The **sprint start and end dates** (format: `MMM D – MMM D, YYYY`, e.g., `May 5 – May 16, 2026`)

These are used for the report title.

### Step 8: Save and Publish to Confluence

1. **Save locally** — write the report to `~/sprint-bug-reports/defect-management-report-[TEAM]-[sprint-id]-[date].md` where TEAM is `OnbrdInsight`, `CNM`, or `SWS`

2. **Upload dashboard** — attach the generated PNG to the Confluence page, then embed it at the top (after the title/sprint metadata) with a `### 📊 TL;DR — Sprint at a Glance` heading above it, using `<ac:image ac:width="760"><ri:attachment ri:filename="..."/></ac:image>` in storage format. **Important**: When publishing in markdown format, reference the image using the latest attachment version URL from the upload response (version number + modificationDate). When updating an existing page's dashboard, always re-upload the PNG and update the image reference to the new version URL to bust Confluence's cache.

3. **Publish to Confluence** using `confluence_create_page`:
   - **Space**: `QA`
   - **Parent page ID**:
     - Core Engineering: `5486968870` (CORE Engineering Insight)
     - SWS: `5487198222` (SWS)
   - **Title**: `Defect Management Report - [TEAM] - Sprint [number from Jira] ([sprint start date] – [sprint end date])`
     - Example: `Defect Management Report - OnbrdInsight - Sprint 26.6 (Apr 30 – May 21, 2026)`
     - Example: `Defect Management Report - CNM - Sprint 26.6 (May 8 – May 21, 2026)`
   - **Content format**: `storage` (required for native TOC macro and red font colors)
   - **TOC**: Place `<ac:structured-macro ac:name="toc"><ac:parameter ac:name="minLevel">2</ac:parameter><ac:parameter ac:name="maxLevel">2</ac:parameter><ac:parameter ac:name="exclude">.*Sprint Health.*</ac:parameter></ac:structured-macro>` after the h1 heading and sprint metadata, before the Sprint Health section. This excludes Sprint Health from the TOC.

3. **Confirm** — provide the user with both the local file path and the Confluence page URL.

## Design Principles

This report is intentionally concise. It answers three questions:
1. **How healthy was this sprint?** (Health score + severity distribution)
2. **Where are the problems?** (Component hotspots + open P0/P1s)
3. **What should we do next?** (Recommendations + carry-over risk)

### Formatting Rules

- **Red highlights**: Use `<font color="red">**text**</font>` for critical data points: P0/P1 percentages, carry-over counts, bugs aged 30+ days, and the health summary when Red.
- **Jira hyperlinks**: All issue keys in tables must link to the Jira issue using `[KEY](https://eeroinc.atlassian.net/browse/KEY)` format.
- **JQL hyperlinks on classification tables**: Every row in a classification/breakdown table (severity, bug source, bug reason, component, platform, epic) must include a clickable JQL link that filters exactly that subset. Additionally, the Sprint Health metrics table, Created vs. Resolved table, Bug Age Distribution table, and Sprint-over-Sprint Trend table must also have JQL links on all numeric counts. Use the format `[value](https://eeroinc.atlassian.net/issues/?jql=<URL-encoded-JQL>)` where the JQL combines the sprint filter with the relevant field filter. **Exception**: The Operational Excellence section count should NOT be linked since individual ticket keys are already hyperlinked in the same row. Examples:
  - Priority row: `[10](https://eeroinc.atlassian.net/issues/?jql=sprint%3D15981%20AND%20priority%3D%22P0%20-%20Blocker%22%20AND%20issuetype%20in%20(Bug%2C%20Defect))`
  - Bug Source row: `[15](https://eeroinc.atlassian.net/issues/?jql=sprint%3D15981%20AND%20cf[11594]%3D%22Manual%20Testing%22%20AND%20issuetype%20in%20(Bug%2C%20Defect))`
  - Component row: `[8](https://eeroinc.atlassian.net/issues/?jql=sprint%3D15981%20AND%20component%3D%22iOS%22%20AND%20issuetype%20in%20(Bug%2C%20Defect))`
  - Bug Reason row: `[12](https://eeroinc.atlassian.net/issues/?jql=sprint%3D15981%20AND%20cf[11560]%3D%22Incorrect%20Logic%22%20AND%20issuetype%20in%20(Bug%2C%20Defect))`
  - Platform row: `[7](https://eeroinc.atlassian.net/issues/?jql=sprint%3D15981%20AND%20cf[11809]%3D%22iOS%22%20AND%20issuetype%20in%20(Bug%2C%20Defect))`
  The count value in the "Sprint [current]" column becomes the hyperlink text. Previous sprint counts should also link using the previous sprint ID. This allows readers to click any number and immediately see the exact tickets in Jira.
- **Epic hyperlinks**: In the Bugs by Epic table, the Epic Key column must link directly to the epic: `[EPIC-KEY](https://eeroinc.atlassian.net/browse/EPIC-KEY)`.
- **Bold**: P0 and P1 rows in severity tables, top component names, and carry-over impact statement.
- **Days Open > 60**: Highlight in red to flag stale critical bugs.

- **No emoji on Decisions header**: The "Decisions & Recommendations" heading must NOT include the 📋 emoji — keep it plain text for clean TOC rendering.
- **Dashboard placement**: The TL;DR dashboard image must appear ABOVE the TOC macro, not below it.

Avoid: lengthy root cause narratives, per-bug analysis, historical deep-dives. Those belong in the Bug Escape Analysis skill.

## Workflow Summary

1. User requests sprint bug health report
2. Skill asks which project (Core or SWS) and requires sprint ID
3. Skill fetches bugs via `jira_search`
4. QAE agent analyzes severity, status, components, filing rate, resolution rate
5. SDM agent produces health score, recommendations, and carry-over assessment
6. Skill composes the report
7. Skill retrieves sprint name and date range from Jira
8. Skill saves to `~/sprint-bug-reports/` (as `defect-management-report-[project]-[sprint-id]-[date].md`) and publishes to Confluence under `QA > Defect Management Metrics`
9. Skill confirms file path and Confluence URL to the user

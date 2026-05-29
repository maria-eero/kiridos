# On-Call Report Generator

Use when asked to generate the automation on-call handoff report. This skill guides the engineer through collecting pipeline data from GitLab, calculating pass rates, pulling related Jira tickets and PRs, then publishes the appropriate report (Routine Monitoring or Maintenance Triage) as a Confluence page.

## Agent

Use the `lead-software-developer-in-test` agent to execute this skill.

## Prerequisites

- `glab` CLI configured with access to the GitLab project (the skill will check and offer setup guidance)
- Confluence MCP (mcp-atlassian) configured for the `QA` space
- Jira MCP (mcp-atlassian) configured for the `QA` project

## Workflow

### Step 1: Check glab Configuration

Ask the engineer:

> Do you have `glab` CLI configured for your GitLab project? If not, I can help you set it up first.

If they need setup, guide them through:
```
glab auth login
```
Then confirm they can run `glab pipeline list` successfully before proceeding.

### Step 2: Collect GitLab Pipeline Data

Ask the engineer:

> Please provide the GitLab project path (e.g., `group/project-name`) or the full URL to the pipelines page.
> Also, whose pipelines and PRs should I filter for? (e.g., branch prefix like `maria/`, GitHub username like `maria-eero`)

Once provided, use `glab api` to pull pipelines for the date range on the `main` branch (these are the scheduled runs that execute the full test suites):
```bash
glab api "projects/<URL-ENCODED-PROJECT>/pipelines?per_page=100&updated_after=<START>T00:00:00Z&updated_before=<END>T00:00:00Z&order_by=id&sort=desc&ref=main" --hostname <GITLAB_HOST>
```

For each pipeline with status `success` or `failed`, fetch the **test report** (not job status):
```bash
glab api "projects/<URL-ENCODED-PROJECT>/pipelines/<PIPELINE_ID>/test_report" --hostname <GITLAB_HOST>
```

The test report contains actual test case results: `total_count`, `success_count`, `failed_count`, `skipped_count`.

**Classify pipelines into suites by total test count:**
- **Sanity suite**: ~51–54 total tests per run
- **Smoke Mobile suite**: ~150–156 total tests per run

**Calculate pass rates excluding skipped tests:**
- Pass rate = `success_count / (success_count + failed_count) × 100`
- Report per-day and overall pass rates for each suite separately

Also fetch the engineer's branch pipelines (filtered by their branch prefix) to report pipeline health:
```bash
glab api "projects/<URL-ENCODED-PROJECT>/pipelines?per_page=100&updated_after=<START>T00:00:00Z&updated_before=<END>T00:00:00Z&order_by=id&sort=desc" --hostname <GITLAB_HOST>
```
Filter results to only branches matching the engineer's prefix (e.g., `maria/`).

### Step 3: Determine Report Type

Based on the calculated overall pass rate:
- **≥95%** → Use **Routine Monitoring Form**
- **<95%** → Use **Maintenance Triage Form**

Inform the engineer which form will be generated and why.

### Step 4: Pull Merge Requests and Related Jira Tickets

Get merged PRs from GitHub for the date range (PRs are on GitHub, not GitLab):
```bash
gh pr list --repo <ORG>/<REPO> --author <GITHUB_USERNAME> --state merged --search "merged:<START>..<END>" --json number,title,headRefName,mergedAt --limit 50
```

Also fetch open PRs to include as carry-over items:
```bash
gh pr list --repo <ORG>/<REPO> --author <GITHUB_USERNAME> --state open --json number,title,headRefName,createdAt --limit 50
```

For each MR:
- Extract the branch name
- Look for Jira ticket patterns in the branch name (e.g., `QA-1234`, `TEAM-567`)
- Query Jira for those tickets to get their summary, status, and type

Also query Jira for recent tickets in the maintenance epic:
```
jira_search: project = QA AND "Epic Link" = QA-13544 AND updated >= -7d
```

And tickets with the `appium-bug` label filed this week:
```
jira_search: project = QA AND labels = appium-bug AND created >= -7d
```

### Step 5: Prompt for Supplemental Information

Ask the engineer for information that cannot be auto-detected:

**Always ask:**
- On-call engineer name
- Any handoff notes or observations for the next on-call
- Any environment issues observed during the week

**If Routine Monitoring Form:**
- Any flaky tests detected (test name, suite, occurrences)
- Confirm if action items exist for next shift or "all clear"

**If Maintenance Triage Form (pass rate <95%), also ask:**
- Current health score and health score from 24h ago
- Metrics report URL (if already generated)
- Root cause hypothesis for failures
- Actions taken to resolve issues
- Whether there's an active release impacted
- Severity assessment (P1-P4) or let the skill auto-classify based on thresholds

### Step 6: Generate and Publish the Report

Compose the report content using the appropriate template, filling in:
- All auto-collected data (pipelines, pass rates, MRs, Jira tickets)
- All engineer-provided supplemental info
- Calculated classifications (pass rate threshold, trend direction, severity)

**Formatting requirements:**
- Add a `👉 **[Jump to Handoff Notes](#handoff-notes)**` anchor link below the Overview table for quick navigation
- Enable heading anchors when publishing (`enable_heading_anchors: true`)
- In the Handoff Notes section, use visual markers for scannability:
  - A blockquote callout (⚠️ or ✅) for overall status
  - 🔴 for urgent items (open PRs to merge, blockers)
  - 🎯 for focus areas
  - ⚡ for action items (numbered list)
  - Bold key phrases within each item

Publish as a new Confluence child page:
- **Space**: `QA`
- **Parent page ID**: `5475172360` (On-call Reports)
- **Title format**: `On-Call Report - [Date Range] - [Engineer Name]`
- **Content format**: markdown

After publishing, provide the engineer with the page URL.

### Step 7: Generate Slack Handoff Message

Ask the engineer:

> What is your Slack username and the next on-call engineer's Slack username? (e.g., `@merebelo` and `@nextperson`)

Then generate a ready-to-copy message for the automation Slack channel:

```
🔄 On-call report handoff from @[current-engineer] to @[next-engineer]

📋 Report: [Confluence page URL]
📊 Test Pass Rates: Sanity [X%] | Smoke Mobile [Y%]
📝 Key notes: [1-2 sentence summary from handoff notes]

Please review before your shift begins. Reach out if you have questions!
```

Present this to the engineer formatted and ready to paste into Slack.

## Template Reference

### Routine Monitoring Form Structure
- Overview (date, time, engineer)
- Test Suite Pass Rates (Sanity suite: per-day table + overall; Smoke Mobile suite: per-day table + overall)
- Pipelines Monitored (table: pipeline ID, branch, pass rate, status, notes)
- Overall Pass Rate + threshold status
- Minor Issues (flaky tests table, environment warnings)
- Handoff Notes (status summary + action items for next shift)

### Maintenance Triage Form Structure
1. Initial Detection & Pass Rate Check (current vs previous, trend, classification)
2. GitLab Failure Investigation (pipeline URL, failed jobs, failure pattern, error analysis, recent changes, root cause)
3. Health Score Assessment (report URL, score comparison table, trend, key insights, combined assessment)
4. Severity Classification (P1-P4 assignment, impact assessment, root cause type)
5. Ticket Report (primary ticket + additional tickets)
6. Resolution (action taken, expected recovery, verification, actual recovery)

## Source Pages
- Guidelines: https://eeroinc.atlassian.net/wiki/spaces/QA/pages/5293441030
- Routine Form: https://eeroinc.atlassian.net/wiki/spaces/QA/pages/5331058726
- Triage Form: https://eeroinc.atlassian.net/wiki/spaces/QA/pages/5296193551
- Reports folder: https://eeroinc.atlassian.net/wiki/spaces/QA/pages/5475172360
- Maintenance Epic: https://eeroinc.atlassian.net/browse/QA-13544

# Dev Completed Summary

An AI-assisted tool that generates Dev Completed ticket summaries for eero mobile releases. It pulls data from Jira and Confluence, then produces a self-contained HTML summary you can open in any browser. This tool aims to help the release leads to have a better overview of the release and help find the owners of tickets that need to be taken care of.

Designed as a **single instructions file** — no scripts, no dependencies to install, no build steps. Just hand the file to an AI assistant and give it a version number.

## Usage

1. Open a conversation with an AI assistant that has Jira and Confluence MCP access (see [Dependencies](#dependencies))
2. Ask: *"Read the file `dev_completed_summary_instructions.md` and generate a Dev Completed Summary for version 26.0.1"*
3. The assistant handles everything else — open the generated HTML file in your browser

💡 **Tip:** If you have GitHub MCP configured, you can reference the instructions file directly from the repo without downloading it. This way you always use the latest version:

> *"Read the instructions at https://github.com/eero-inc/dev-completed-summary/blob/main/dev_completed_summary_instructions.md and generate a Dev Completed Summary for version 26.0.1"*

💡 *You can use it for single versions (26.X), dot releases (26.X.X) and grouped versions (26.X and 26.X.X)*

## What You Get

A single HTML file with:
- All Dev Completed tickets grouped by project (CORE / SWS) and platform (Android / iOS)
- PR merge status and dates
- RC availability — whether each ticket's PR was merged before the RC cutoff
- QA Assignee for each of the tickets
- Links to every Jira ticket and parent epic
- An appendix with links to all data sources (Confluence release pages, Jira boards)

See [`samples/dev_completed_summary_26.2.html`](samples/dev_completed_summary_26.2.html) for an example.

💡 *Want to share it on Slack? Exporting to a PDF file via browser is recommended as it will still retain links information on the file.*

## Files

| File | Description |
|------|-------------|
| `dev_completed_summary_instructions.md` | The instructions file — this is the tool |
| `samples/dev_completed_summary_26.2.html` | Sample output for version 26.2 |

## Dependencies

This tool requires an AI assistant with MCP (Model Context Protocol) access to:

| Dependency | Purpose |
|------------|---------|
| **Jira MCP** | Fetches Dev Completed tickets, PR status, QA assignees, parent epics |
| **Confluence MCP** | Discovers Jira version IDs from release feature pages, fetches RC cutoff dates from release notes |

Any AI assistant that supports MCP tool calling and has these two integrations configured should work (e.g., Kiro CLI, Cline, Claude with MCP servers, etc.).

For MCP setup instructions, see: [Setup Atlassian MCP](https://eeroinc.atlassian.net/wiki/spaces/BI/pages/5146509335/Setup+Atlassian+MCP)
 
## Optional 1: Auto-Approving Tool Calls
 
When running this tool, the AI assistant makes many calls to four read-only MCP endpoints:
- `jira_get_issue` — fetch individual Jira tickets
- `jira_search` — run JQL queries to find Dev Completed tickets
- `confluence_get_page` — retrieve release pages and RC cutoff dates
- `confluence_search` — discover Jira version IDs from Confluence
 
By default, Kiro prompts for confirmation before each tool call. For a workflow that makes dozens of Jira and Confluence lookups, this creates significant friction. You can auto-approve these four read-only tools to let the assistant work without interruption.
 
**To auto-approve**, run this slash command at the start of your Kiro session:

```
/tools trust jira_get_issue jira_search confluence_get_page confluence_search
```

> **Note:** Trust is session-only — it resets when you start a new session.
 
**Why these tools are safe to auto-approve:**
- All four are strictly read-only — they cannot create, modify, or delete any data
- They access data you already have permission to view using your existing credentials
- Jira and Confluence enforce their own access controls; the tools cannot bypass them
 
 
## Optional 2: Confluence Upload

After generating the summary, the assistant can optionally upload it directly to a Confluence page. This is configured in the instructions file and is prompted at the end of each run. Unfortunately, Confluence does not support the same depth of formatting as HTML, so the result will have all the information but in a less polished style.

[Confluence: Dev Completed Summary](https://eeroinc.atlassian.net/wiki/spaces/QA/pages/5309202581)

## How much time will this save me?
 
**Manual process:** ~3 hours per report
- Navigating between Jira tickets, PRs information, and Confluence pages
- Cross-referencing RC cutoff dates
- Copying data into a spreadsheet or document
- Formatting, organizing and sharing the final report with the team
 
**With this tool:** ~10-15 minutes per report
- 30 seconds to provide the version number
- 2-3 minutes for the AI to gather all data
- 5-10 minutes to review the generated HTML
 
**Time saved per report: ~3 hours** (90-95% reduction)
 
<details>
<summary><strong>Manual process breakdown (per ~30 tickets)</strong></summary>
 
- **Per ticket:** 2-4 minutes (3.5 min average)
  - Navigate to Jira board, find Dev Completed ticket (15-30s)
  - Open ticket, check status and summary (10-15s)
  - Find and open PR information (10-20s)
  - Check PR merge status and date (15-30s)
  - Navigate to parent epic (20-30s)
  - Check QA assignee (5-10s)
  - Cross-reference RC cutoff date in Confluence (30-60s)
  - Copy data to spreadsheet/doc (20-30s)
- **All tickets:** ~105 minutes (30 tickets × 3.5 min average)
- **Setup and formatting:** ~65 minutes
  - Initial setup (open boards, create doc structure): 20 min
  - Context switching between tools: 15 min
  - Final formatting and organization: 30-45 min
- **Total:** ~3 hours
 
</details>
 
### Current Reality vs. What's Possible
 
**Today:** Due to time constraints and competing priorities, release leads typically generate this report only during the final week before production release. This means issues are discovered late, leaving less time to address them.
 
**With this tool:** The 10-15 minute runtime makes it practical to generate the report **every week** throughout the 3-week release cycle. Teams can identify blockers, missing PRs, or QA assignment gaps much earlier, giving more time to resolve issues before the production deadline.
 
**Impact:**
- **Per 3-week release:** ~6 hours saved (if running twice per release)
- **Bonus:** Earlier visibility into release health → fewer last-minute surprises

## License

Internal eero project - All rights reserved

---

**Created and Maintained by**: [@rodrigo-eero](https://github.com/rodrigo-eero)

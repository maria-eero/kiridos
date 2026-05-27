---
name: project-setup
description: Use when setting up a new project folder. Gathers info from attached documents, Confluence pages, and Jira tickets to generate organized project documentation.
---

# Project Setup Skill

Create a project folder in the workspace from attached documents, Confluence pages, and Jira tickets. Produces organized markdown files covering architecture, team, environments, processes, and product context.

## Inputs

The user provides:

1. **Project name** — used as the folder name (kebab-case)
2. **Attached documents** (optional) — any files relevant to the project
3. **Confluence page URL** (optional) — project page to fetch
4. **Jira ticket keys** (optional) — initiative/epic keys to crawl

## Authentication

Source credentials from `~/.zshrc`:

```bash
source ~/.zshrc
# Available vars: JIRA_EMAIL, JIRA_API_TOKEN
```

All API calls use **Basic Auth**: `-u "$JIRA_EMAIL:$JIRA_API_TOKEN"`

## Fetching from Confluence

```bash
# Extract page ID from URL (the numeric part in /pages/{id}/...)
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  "https://eeroinc.atlassian.net/wiki/rest/api/content/{pageId}?expand=body.storage,children.page"
```

- Parse the HTML body to extract project info (overview, decisions, team, links)
- If the page has child pages, fetch those too

## Fetching from Jira

### Get a single ticket

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  "https://eeroinc.atlassian.net/rest/api/2/issue/{KEY}"
```

### Search for child tickets (v3 API — required)

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "https://eeroinc.atlassian.net/rest/api/3/search/jql" \
  -d '{"jql":"parent={KEY} ORDER BY created ASC","maxResults":50,"fields":["key","summary","status","issuetype","assignee","parent"]}'
```

> ⚠️ The v2 `/rest/api/2/search` endpoint is **deprecated and returns errors**. Always use the v3 POST endpoint above.

### Crawling hierarchy

For an **Initiative**, the hierarchy is:
```
Initiative (e.g. SWS-31406)
  └── Epics (parent=SWS-31406)
       └── Stories/Tasks (parent={epic-key})
```

1. Fetch the initiative ticket for overview
2. Search `parent={initiative-key}` to get epics
3. Search `parent in ({epic1}, {epic2}, ...)` to get all stories/tasks

## Output Structure

Create a folder named after the project under the workspace root:

```
{project-name}/
├── README.md
├── architecture.md
├── team.md
├── environments.md
├── processes.md
├── product.md
└── tickets.md
```

### File Descriptions

| File | Content |
|------|---------|
| `README.md` | Project overview, purpose, key links, and index of folder contents |
| `architecture.md` | System design, tech stack, integrations, API specs, data flow |
| `team.md` | Team members, roles, responsibilities (extracted from assignees + page mentions) |
| `environments.md` | Environment URLs, feature flags, deployment targets, access info |
| `processes.md` | Workflows, ceremonies, decisions log, conventions |
| `product.md` | Features, business rules, user stories, Figma links, requirements |
| `tickets.md` | Jira tickets organized by epic — key, summary, status, assignee |

## Rules

- **Only create files that have content** — skip any category with no relevant information
- **Keep content faithful** to source documents — don't invent information
- **Format cleanly** in markdown with proper headings, lists, and tables
- **Deduplicate** — if the same info appears in multiple sources, consolidate it
- **Link back** — include Jira ticket keys and Confluence page URLs as references
- **Extract meeting notes** — Confluence pages often have expandable sync notes; include key decisions and status updates in `processes.md`

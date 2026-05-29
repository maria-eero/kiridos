# QA Process Planning Skill

Generates three QA artifacts from project documentation (tech specs, Figma designs, PRDs).

## Artifacts

| # | Artifact | Format |
|---|----------|--------|
| 1 | Quality Process Flowchart | Mermaid diagram |
| 2 | Test Plan (v3.1 template) | Markdown |
| 3 | Automation Plan | Markdown |

## How It Works

1. Provide project docs (tech spec, Figma, PRD — any combination)
2. Skill asks clarifying questions and confirms platform (Mobile/Web/Both)
3. Generates all 3 artifacts using specialized agents (Lead QAE + Lead SDET)
4. SDM agent reviews for feasibility, resourcing, and risks
5. Revisions applied → final versions produced
6. Optional: creates a Jira epic with stories/subtasks derived from the plans

## Usage

```
I need to create QA process artifacts for our new [Feature Name] project.

Here's what I have:
- Tech spec: [link or paste]
- Figma designs: [link]
- PRD: [link or paste]
- Platform: Mobile (iOS + Android)

Please generate the QA process flowchart, test plan, and automation plan.
```

## Artifact Summary

**Flowchart** — Mermaid diagram showing the end-to-end QA process with decision points, parallel tracks, and quality gates. Renders in Amazon Wiki or Harmony Mermaid Editor.

**Test Plan** — Follows our v3.1 template. Covers scope, environment, test data, strategy, scenarios mapped to requirements, risks, and sign-off. Applies the full review checklist (hardware, soak, Plus, setup, UX, i18n, ISP, parity, T-mobile).

**Automation Plan** — Scope decisions with justification, strategy by layer (Unit → Integration → API → E2E), Figma-to-test mapping, phased implementation, CI/CD integration, and maintenance strategy.

## When to Use

- New feature kicks off and you need QA planning artifacts
- Major scope change requires test plan updates
- Assessing automation feasibility for a feature
- Need a structured starting point to refine with the team

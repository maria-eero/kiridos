# Team

Roles below are inferred from document authorship and Jira ticket assignees.

| Role | Name |
|------|------|
| Cloud / Auth lead (ERD + HLD author) | Burak Varan |
| Cloud Engineer (AuthX / AuthPortal config) | Harry Posner |
| Cloud Engineer (deletion, CES, sync) | Jonathan Muniz-Murguia |
| Mobile Engineer (Mobile Tech Spec author, Android PoC) | Adauton Heringer |
| Insight & account.eero.com | Felipe Ricardi |
| PBR | Ryan Thompson |
| QA | Henrique |

## Responsibilities

- **Cloud / Passwordless (CORE-28017):** Burak Varan (lead), Harry Posner, Jonathan Muniz-Murguia
- **Mobile (CORE-32410):** Adauton Heringer
- **Insight & account.eero.com (CORE-32858):** Felipe Ricardi
- **QA:** Henrique

## Communication

- **Slack:** [#proj channel](https://eero.slack.com/archives/C0AA52DR28G)
- **Security consult:** [SEC-2470](https://eeroinc.atlassian.net/browse/SEC-2470)

## External Dependencies

- **Amazon Identity / AuthX team** — passwordless (EOA/MOA) for PAPs, split-page sign-in, email passwordless, migration APIs, UnifiedCX, marketing-consent checkbox, Irish/Icelandic locale.
- **External Partners (Frontier)** — verified user creation via `POST /2.2/organizations/self/register` must keep working; ongoing PAP account creation.

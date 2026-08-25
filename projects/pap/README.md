# PAP Migration (Private Account Pool)

## Overview

The PAP Migration moves eero's ~14M consumer "eero auth" users off eero's self-managed OTP authentication system and onto an **Amazon Identity Private Account Pool (PAP)**, authenticated through **Amazon AuthPortal**. Clients authenticate against the eero PAP (via the MAP SDK on mobile and AuthPortal web views on web), receive a MAP token, and exchange it with eero cloud at `POST /2.3/login` for an eero user session.

The migration has two parts:

1. **Enable passwordless auth for PAPs** (Identity/AuthX team): passwordless account creation and OTP sign-in (phone = MOA, email = EOA, both = MCA) as reusable PAP capabilities.
2. **Migrate eero auth users to the PAP** and transition all client surfaces (iOS, Android, Insight, account.eero.com) to AuthPortal for registration and login.

**In scope:** eero auth users only (`authenticationType = EeroAuthentication` — no `amazonCustomerId`, no `identityProviderId`).
**Out of scope:** Amazon-login users, SSO users, password/passkey support at launch, 3P login.

## Status

**Ready for Implementation** (HLD dated Jun 4, 2026; ERD dated Apr 17, 2026). Cloud passwordless work (CORE-28017) is In Execution. Mobile support (CORE-32410) and Insight/web (CORE-32858) are at Idea/Spike stage.

> ⚠️ **QA note — test plan not yet started.** There is an open concern about how automated and manual test-user login will work under the new PAP/AuthPortal/MAP flow (headless OTP retrieval is no longer available). See **[test-user-login-concerns.md](./test-user-login-concerns.md)** before writing the test plan.

## Key Links

- **Initiative:** [INIT-351 — Private Account Pool (PAP)](https://eeroinc.atlassian.net/browse/INIT-351)
- **Mobile Epic:** [CORE-32410 — PAP - Mobile Support](https://eeroinc.atlassian.net/browse/CORE-32410)
- **Cloud/Passwordless Epic:** [CORE-28017 — Support Passwordless login in PAP](https://eeroinc.atlassian.net/browse/CORE-28017)
- **Insight & account.eero.com Epic:** [CORE-32858](https://eeroinc.atlassian.net/browse/CORE-32858)
- **Mobile Tech Spec:** [PAP Migration - Mobile Tech Spec](https://docs.google.com/document/d/1yZ3M5ze90yTmJ3I15H0F2co84VP6EloHuWBLNR28zrg/edit) (author: Adauton Heringer)
- **Cloud HLD:** [PAP Migration HLD](https://docs.google.com/document/d/1tllKzECjAGPHwXNzsTnGXMsy8FusrG1WC98eSnV4pis/edit) (author: Burak Varan)
- **ERD:** [PAP for eero Auth Users — ERD](https://docs.google.com/document/d/1mGoR0uoGjoKeqOD5w3BS-kvnyluwjPMsI-qC91rXLUQ/edit)
- **SSO Figma:** [SSO / Welcome screen](https://www.figma.com/design/WjOfX2phnwdS2Go0iPDNSY/SSO?node-id=738-1863)
- **Slack:** [#proj channel](https://eero.slack.com/archives/C0AA52DR28G)
- **Security Consult:** [SEC-2470](https://eeroinc.atlassian.net/browse/SEC-2470)
- **PBR:** [PBR-58](https://eeroinc.atlassian.net/browse/PBR-58)

## Project Documents

| Document | Description |
|----------|-------------|
| [README.md](./README.md) | This file — project overview |
| [architecture.md](./architecture.md) | Auth flows, endpoints, components, data model, feature flags |
| [product.md](./product.md) | Requirements, UX (Welcome screen, UnifiedCX), migration options |
| [processes.md](./processes.md) | Migration phases, rollout/rollback, release strategy, key decisions |
| [team.md](./team.md) | Team members and roles |
| [tickets.md](./tickets.md) | Jira initiative / epics / task breakdown |
| [screens.md](./screens.md) | Figma screen reference (Welcome V2, PAP AuthPortal sign-in/create/verify flow) |
| [open-questions.md](./open-questions.md) | Open questions and TBDs for team syncs |
| [test-user-login-concerns.md](./test-user-login-concerns.md) | **QA blocker analysis** — how test users log in under PAP/MAP |
| [references/PAP-Migration-Mobile-Tech-Spec.md](./references/PAP-Migration-Mobile-Tech-Spec.md) | Local copy of the Mobile Tech Spec (Adauton Heringer) — source for architecture/product docs |
| [references/PAP-Amazon-Identity-Testing-Guide.md](./references/PAP-Amazon-Identity-Testing-Guide.md) | PAP ↔ Amazon Identity testing guide |

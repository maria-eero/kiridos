# Jira Tickets

## Initiative

| Key | Summary | Status | Priority |
|-----|---------|--------|----------|
| [INIT-351](https://eeroinc.atlassian.net/browse/INIT-351) | Private Account Pool (PAP) | Opportunity | P3 - Minor |

## Epics

| Key | Summary | Status | Assignee | Priority |
|-----|---------|--------|----------|----------|
| [CORE-28017](https://eeroinc.atlassian.net/browse/CORE-28017) | Amazon PAP - Support Passwordless login in Private Account pool | In Execution | Burak Varan | P1 - Critical |
| [CORE-32410](https://eeroinc.atlassian.net/browse/CORE-32410) | PAP - Mobile Support | Idea | Adauton Heringer | P3 - Minor |
| [CORE-32858](https://eeroinc.atlassian.net/browse/CORE-32858) | PAP - Insight and account.eero.com Support | Idea | Felipe Ricardi | P3 - Minor |

## [CORE-32410] PAP - Mobile Support

| Key | Type | Summary | Status | Assignee |
|-----|------|---------|--------|----------|
| [CORE-31992](https://eeroinc.atlassian.net/browse/CORE-31992) | Task | [Mobile] Create PAP migration mobile tech spec | Dev In Progress | Adauton Heringer |
| [CORE-32351](https://eeroinc.atlassian.net/browse/CORE-32351) | Task | [Android] Create PAP PoC | Done | Adauton Heringer |
| [CORE-32807](https://eeroinc.atlassian.net/browse/CORE-32807) | Task | [Android] Implement force update screen for PAP | Dev Completed | Adauton Heringer |

## [CORE-32858] PAP - Insight & account.eero.com Support

| Key | Type | Summary | Status | Assignee |
|-----|------|---------|--------|----------|
| [CORE-32963](https://eeroinc.atlassian.net/browse/CORE-32963) | Spike | [PAP] HLD + Jira tasks | Dev In Progress | Felipe Ricardi |
| [CORE-33198](https://eeroinc.atlassian.net/browse/CORE-33198) | Spike | [PAP] Proof of concept for accounts.eero.com | Backlog | Felipe Ricardi |

## [CORE-28017] Support Passwordless login in PAP (Cloud) — selected tasks

Highlights of the 70 child tasks (full list in Jira). **QA-relevant items in bold.**

| Key | Summary | Status | Assignee |
|-----|---------|--------|----------|
| [CORE-29339](https://eeroinc.atlassian.net/browse/CORE-29339) | Create ERD | Done | Burak Varan |
| [CORE-30257](https://eeroinc.atlassian.net/browse/CORE-30257) | Create HLD for migration (exclude passwordless MOA+EOA impl) | Done | Burak Varan |
| [CORE-30258](https://eeroinc.atlassian.net/browse/CORE-30258) | Feature flag/throttle for "upgrade your app" messaging (feature_204) | Done | Burak Varan |
| [CORE-30259](https://eeroinc.atlassian.net/browse/CORE-30259) | Feature flag/throttle for switching to AuthX login+registration (feature_203) | Done | Burak Varan |
| [CORE-30260](https://eeroinc.atlassian.net/browse/CORE-30260) | New 4XX error: client does not support AuthX | Done | Burak Varan |
| [CORE-30985](https://eeroinc.atlassian.net/browse/CORE-30985) | Add pap_customer_id and pap_linked_at columns | Done | Burak Varan |
| [CORE-30988](https://eeroinc.atlassian.net/browse/CORE-30988) | Implement CreateAccountV2 client in authenticationlib | Done | Burak Varan |
| [CORE-30989](https://eeroinc.atlassian.net/browse/CORE-30989) | Implement SetEmail, SetMobile clients | Done | Burak Varan |
| [CORE-32370](https://eeroinc.atlassian.net/browse/CORE-32370) | Enable EOA account creation for non-retail PAP by association handle | Done | Harry Posner |
| [CORE-32374](https://eeroinc.atlassian.net/browse/CORE-32374) | Aztec verifyAccessTokenForPermissions audit | Done | Burak Varan |
| [CORE-32375](https://eeroinc.atlassian.net/browse/CORE-32375) | Confirm Aztec/Address paths for PAP marketplace | Done | Burak Varan |
| [CORE-32376](https://eeroinc.atlassian.net/browse/CORE-32376) | Implement PAP login route POST /2.3/login | Done | Burak Varan |
| [CORE-32378](https://eeroinc.atlassian.net/browse/CORE-32378) | Add is_pro handling to /2.3/login | Backlog | Burak Varan |
| [CORE-32698](https://eeroinc.atlassian.net/browse/CORE-32698) | Enable UnifiedAuth for eero domains | Done | Harry Posner |
| [CORE-32872](https://eeroinc.atlassian.net/browse/CORE-32872) | Synchronous PAP account creation on registration routes | Backlog | Burak Varan |
| [CORE-32873](https://eeroinc.atlassian.net/browse/CORE-32873) | CES consumer implementation in monolithconsumers | Triage | Harry Posner |
| [CORE-32874](https://eeroinc.atlassian.net/browse/CORE-32874) | Implement migration APIs | Backlog | Burak Varan |
| [CORE-32875](https://eeroinc.atlassian.net/browse/CORE-32875) | Implement reconciliation APIs | Backlog | Burak Varan |
| [CORE-32878](https://eeroinc.atlassian.net/browse/CORE-32878) | Sandfire deletion workflow onboarding | Dev In Progress | Jonathan Muniz-Murguia |
| [CORE-32879](https://eeroinc.atlassian.net/browse/CORE-32879) | Add account deletion API client | In Pull Request | Jonathan Muniz-Murguia |
| [CORE-32882](https://eeroinc.atlassian.net/browse/CORE-32882) | CES Odin material set access + SNS subscription permission | In Pull Request | Harry Posner |
| [CORE-32883](https://eeroinc.atlassian.net/browse/CORE-32883) | CES sync reliability: DLQ, retries, monitoring | Triage | Harry Posner |
| [CORE-32885](https://eeroinc.atlassian.net/browse/CORE-32885) | Account recovery flow for PAP users | Triage | — |
| [CORE-32886](https://eeroinc.atlassian.net/browse/CORE-32886) | Org user invite flow: create PAP account on invite acceptance | Triage | Burak Varan |
| **[CORE-32890](https://eeroinc.atlassian.net/browse/CORE-32890)** | **Investigate Kamino test accounts** | **Triage** | **Unassigned** |
| [CORE-32951](https://eeroinc.atlassian.net/browse/CORE-32951) | Delete PAP customer ID column on ForgetMe | In Pull Request | Jonathan Muniz-Murguia |
| [CORE-33180](https://eeroinc.atlassian.net/browse/CORE-33180) | Handle duplicate account detection for Amazon login | Triage | Burak Varan |
| [PBR-58](https://eeroinc.atlassian.net/browse/PBR-58) | PBR request for "PAP for eero Auth Users" project | In Progress | Ryan Thompson |

> **[CORE-32890 — Investigate Kamino test accounts]** is the ticket that directly gates QA automation. It is currently in Triage and unassigned. See [test-user-login-concerns.md](./test-user-login-concerns.md).

## Notable "Won't Do" / dropped

- CORE-30990 (DeleteAccountV2 client) — Won't Do; superseded by account deletion API client (CORE-32879).
- CORE-30994/30995/30996/30997 (allowlist domain, Stego config, weblab bypass) — Won't Do.
- CORE-32379 (Backfill admin API route + MCM) — Won't Do; migration APIs tracked under CORE-32874/32876.

## Status Snapshot (Mobile epic)

| Status | Count |
|--------|-------|
| Done | 1 |
| Dev Completed | 1 |
| Dev In Progress | 1 |
| **Total** | **3** |

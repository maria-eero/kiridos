# PAP — Open Questions

## 🔴 QA Blocker — Test-User Login (resolve before test plan)

See **[test-user-login-concerns.md](./test-user-login-concerns.md)** for the full analysis. Summary of what must be answered:

| # | Question | Status | Owner / Ticket |
|---|----------|--------|----------------|
| Q-QA-1 | Does **Kamino + APEX device registration** produce valid MAP tokens for the **eero PAP marketplace**? (Enables headless `/2.3/login` automation.) | 🔴 Pending validation — explicit HLD blocker | [CORE-32890](https://eeroinc.atlassian.net/browse/CORE-32890) (Triage, unassigned) + Phase-2 PoC |
| Q-QA-2 | Can we create **devo PAP test accounts** (hardcoded OTP `112233`)? AbeBooks precedent flagged no subsidiary-locale path for devo PAP accounts. | 🔴 TBD | Identity / QA |
| Q-QA-3 | Which environment do RC validation and current test users run against (devo vs stage vs prod)? | 🔴 TBD | QA |
| Q-QA-4 | Do existing automation test users (cloud_smoke, network_admin, mobile_smoke, die_hard, …) migrate to the PAP and vend MAP tokens, or do we provision new Kamino-backed accounts? | 🔴 TBD | QA / Cloud |
| Q-QA-5 | Mobile UI automation (nova-act on Device Farm) through AuthPortal — how is the OTP challenge handled without headless retrieval? | 🔴 TBD | QA / Mobile |
| Q-QA-6 | How are `is_pro`/org and Frontier partner test users authenticated for testing (no PAP equivalent to headless `/2.2/pro/login`)? | 🔴 TBD | Cloud |
| Q-QA-7 | Who owns test-account setup and what's the onboarding lead time? (Kamino ~3 days; PAP-subsidiary onboarding unproven.) | 🔴 TBD | QA / Identity |

## Product / Design

| # | Question | Status | Context |
|---|----------|--------|---------|
| P1 | UnifiedCX vs separate sign-up/sign-in? | TBD (target June 15) | PAP uses UnifiedCX by default; affects duplicate-account detection + mobile UX. Separate paths need extra Identity work. |
| P2 | Which migration option — hard cutover (Option 1) vs gradual sunset (Option 2)? | TBD | HLD leans toward throttle-gated cutover; both documented in product.md. |
| P3 | Marketing consent — extra optional AuthPortal checkbox, or post-registration opt-in in account settings? | TBD | Identity supports one required checkbox (ToS); marketing needs Identity work. |

## Identity / AuthX Dependencies

| # | Question | Status | Target |
|---|----------|--------|--------|
| I1 | Passwordless (EOA/MOA) support for PAPs | In progress | Nov 2026 |
| I2 | CreateAccountV2 indefinite access, authZ scoped to eero PAP, for partner user creation | API exists; authZ scoped confirmed; indefinite access pending L8 alignment | June 15 |
| I3 | CES subscription permission + Odin decryption key access for eero PAP | API exists; permissions not yet requested | TBD |
| I4 | eero marketplace ID creation | In progress | TBD |
| I5 | UnifiedCX PoC | In progress (Identity) | June 15 |
| I6 | Marketing-consent optional checkbox | Needs Identity work | TBD |
| I7 | Irish / Icelandic locale support (~6,000 users) | Panther onboarding via AuthX | TBD |
| I8 | AuthPortal proxy config — target hostname, required headers, TLS/cert | Not started; need details from Identity | TBD |

## Security & Policy (passwordless-only PAP)

The PAP security policy mandates four controls; several may not apply to a passwordless-only pool (confirm with Identity PM):

| # | Question | Status |
|---|----------|--------|
| S1 | **Forced password reset** — no passwords to reset; prerequisite for Not-Me/TIV. Can it be deferred until password support? | Likely deferred |
| S2 | **Not-Me broadcast** — limited value for single-claim passwordless (attacker controlling the OTP channel also gets the notification). | Value depends on account type |
| S3 | **TIV approval** — credential-revocation mechanism doesn't apply to passwordless; does enforcement work differently? | TBD |
| S4 | **Email verification** — applies; eero verifies email during registration. | Confirmed applicable |

## Engineering / Migration

| # | Question | Status |
|---|----------|--------|
| E1 | CreateAccountV2 rate limits for backfill | TBD empirically in preprod |
| E2 | Partner-created users trust model — pre-verified PAP accounts vs just-in-time migration at first AuthPortal login (Q2 in ERD) | Leaning pre-verified with tightened authZ |
| E3 | SMS billing model post-migration (Amazon Identity vs Twilio) — subsidiary billing | Flagged by Identity; IMDB may be reference |
| E4 | Users-table sync mechanism post-migration — AddressService reads vs CES notifications | CES chosen; AddressService fallback |
| E5 | CS tooling post-migration (OTP verification) — keep Twilio (P1) or Identity equivalent | Decoupled; keep Twilio for now, address as P1 |

## Resolved (for reference)

- **Q5 (ERD): Web view vs MAP SDK for mobile** → Decided: **MAP for mobile, AuthPortal for web** (MAP uses web views internally).
- **Q7 (ERD): CS tooling** → CS verification decoupled from migration; keep Twilio for now (P1).
- **Q8 (ERD): Consent collection** → Identity supports one required checkbox (ToS); marketing via extra optional checkbox or post-reg opt-in.

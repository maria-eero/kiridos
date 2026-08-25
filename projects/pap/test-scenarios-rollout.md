# PAP — Test Scenario Matrix (Rollout States)

Derived from the Mobile Tech Spec rollout timeline & rollback strategy (see [architecture.md → Rollout Timeline & Rollback](./architecture.md#rollout-timeline--rollback)). This matrix enumerates the **reachable** combinations of app version, feature flags, and legacy-endpoint state, and the behavior QA must assert in each. It is the seed for the full mobile test plan (scenario section).

> **Testability caveat.** PAP sign-in goes through Amazon AuthPortal OTP, and headless OTP retrieval is not available. Most PAP-login cells below depend on the Kamino test-account path ([CORE-32890](https://eeroinc.atlassian.net/browse/CORE-32890), in Triage) before they can be automated. See [test-user-login-concerns.md](./test-user-login-concerns.md). Automation column reflects candidacy **assuming** that blocker is resolved.

## Dimensions

| Dimension | Values | Source of truth |
|-----------|--------|-----------------|
| **App version band** | `26.10–26.11` (pre-PAP; ships `feature_204` accessor) · `26.12+` (PAP-complete; `papLoginRequiredCapable`) | client build |
| **`feature_203` (papLoginRequired)** | Not enrolled · Enrolled (device in throttle %) | `GET /2.2/app_configuration`, device-ID `HASH_SHARD` throttle |
| **`feature_204` (forceAppUpgradeToUsePap)** | Off · On | `GET /2.2/app_configuration` |
| **Legacy endpoints** | Live · Disabled→404 (`eeroAuthLoginDisabled` / `eeroAuthRegistrationDisabled` DKVs) | cloud DKV |
| **Auth entry** | Amazon sign-in (retail) · PAP Email · PAP Phone · Technicians (SSO) | Welcome screen |
| **User type** | Existing eero-auth (backfilled, `pap_customer_id` set) · New · Org (`is_pro=true`) | `users` table / Identity |

Flag enrollment is per **device** (`X-Client-Device-Id` = IDFV on iOS, app-scoped UUID on Android), so enrollment must be stable across sessions for a given device.

## Rollout-state scenarios

Legend — **Welcome:** V1 = legacy (`WelcomeFragment`/`WelcomeView` + bottom-sheet pickers), V2 = new (`WelcomeFragmentV2`/`WelcomeViewV2`, 3 entries). **Endpoint:** which route the auth path hits.

### State 1 — Pre-rollout (26.12 shipped, both flags OFF, legacy live)

| ID | App | Auth entry | Welcome | Expected behavior | Endpoint | Automation |
|----|-----|-----------|---------|-------------------|----------|-----------|
| ROLL-01 | 26.12+ | Email/Phone | V1 | Legacy eero-auth OTP flow; no PAP; unaffected by shipped `/2.3/login` code | `/2.2/login` · `/2.2/register` | Candidate |
| ROLL-02 | 26.12+ | Amazon sign-in | V1 | Retail Amazon-login unchanged | `/2.2/login/amazon` | Candidate |
| ROLL-03 | 26.10–26.11 | Email/Phone | V1 | Legacy path; `feature_203` invisible (below min-version) | `/2.2/login` · `/2.2/register` | Candidate |

### State 2 — Phased PAP ramp (`feature_203` at X%, `feature_204` OFF, legacy live)

| ID | App / enrollment | Auth entry | Welcome | Expected behavior | Endpoint | Automation |
|----|------------------|-----------|---------|-------------------|----------|-----------|
| ROLL-04 | 26.12+ enrolled | PAP Email | V2 | AuthPortal (`ap.account.eero.com`, handle `amzn_eero_mobile_us`) → MAP token → session | `/2.3/login` | Candidate* |
| ROLL-05 | 26.12+ enrolled | PAP Phone | V2 | AuthPortal phone OTP → MAP token → session | `/2.3/login` | Candidate* |
| ROLL-06 | 26.12+ enrolled | Amazon sign-in | V2 | Retail handle (`amzn_eero_mobile_android_us`), global MAP config unchanged | `/2.2/login/amazon` | Candidate |
| ROLL-07 | 26.12+ enrolled | Technicians | V2 | Routes to SSO (`SsoOtherPartnerFragment` / `SSOCoordinator`), no PAP | SSO | Candidate |
| ROLL-08 | 26.12+ **not** enrolled | Email/Phone | V1 | Falls to legacy eero-auth; V2 never rendered | `/2.2/login` · `/2.2/register` | Candidate |
| ROLL-09 | 26.10–26.11 | Email/Phone | V1 | Legacy path (flag invisible); no update prompt yet (`feature_204` off) | `/2.2/login` · `/2.2/register` | Candidate |
| ROLL-10 | 26.12+ enrolled | PAP Email/Phone — **existing** user | V2 | Resolves to existing `users` row by `pap_customer_id`; `is_new_user=false`; verified session | `/2.3/login` | Candidate* |
| ROLL-11 | 26.12+ enrolled | PAP Email/Phone — **new** user | V2 | New `users` row + `pap_linked_at`; `is_new_user=true` → marketing-consent fallback prompt | `/2.3/login` | Candidate* |
| ROLL-12 | 26.12+ enrolled | PAP — **org** (`is_pro=true`) | V2 | Org membership + IP allowlist enforced; Insight-enabled session | `/2.3/login` | Candidate* |

### State 3 — Cutover trigger (`feature_203` 100%, `feature_204` ON, legacy live)

| ID | App | Auth entry | Welcome | Expected behavior | Endpoint | Automation |
|----|-----|-----------|---------|-------------------|----------|-----------|
| ROLL-13 | 26.12+ | Email/Phone | V2 | Always PAP; update check never reached | `/2.3/login` | Candidate* |
| ROLL-14 | 26.10–26.11 | Email/Phone | V1 | **Update-required screen** (`UpdateRequiredFragment` / `ErrorViewController`) shown *before* credential entry; App Store button | none (blocked) | Candidate |
| ROLL-15 | 26.10–26.11 | Amazon sign-in | V1 | Retail Amazon-login **not** blocked by `feature_204` (prompt is on eero-auth path only) | `/2.2/login/amazon` | Candidate |

### State 4 — Legacy decommissioned (`eeroAuth*Disabled` DKVs flipped → 404)

| ID | App | Auth entry | Expected behavior | Endpoint | Automation |
|----|-----|-----------|-------------------|----------|-----------|
| ROLL-16 | 26.12+ | PAP Email/Phone | Unaffected — PAP path continues | `/2.3/login` → 200 | Candidate* |
| ROLL-17 | 26.10–26.11 (ignored update) | Email/Phone | Raw 404 backstop; `error.login.upgrade_required` | `/2.2/login` · `/2.2/register` → 404 | Candidate |
| ROLL-18 | any | Legacy login/register/pro-login direct | All legacy routes return 404 | `/2.2/login` · `/2.2/pro/login` · `/2.2/register` → 404 | Candidate |

### Rollback scenarios

| ID | Trigger | Expected behavior | Automation |
|----|---------|-------------------|-----------|
| ROLL-19 | `feature_203` throttle → 0% mid-ramp | On next `app_configuration` refresh, 26.12+ enrolled users fall back to V1 + eero-auth; legacy endpoints still serve; no user stranded | Candidate |
| ROLL-20 | `feature_204` → off after premature fire | 26.10–26.11 users stop seeing update prompt on next `app_configuration` refresh | Candidate |

## Edge / cross-cutting scenarios

| ID | Scenario | Expected behavior | Automation |
|----|----------|-------------------|-----------|
| ROLL-21 | `feature_203` toggled live (iOS `@SharedReader` shared state) | Welcome screen + Technicians toolbar react without app relaunch | Candidate |
| ROLL-22 | Device-ID bucketing stability | Same `X-Client-Device-Id` stays consistently enrolled / not-enrolled across sessions (`HASH_SHARD`) | Candidate |
| ROLL-23 | Duplicate-account detection | PAP new-account email/phone matches existing Amazon-login user → `DeleteAccountV2` + `error.form.email.unavailable` / `error.form.phone.unavailable` | Candidate* |
| ROLL-24 | Token encoding | `auth_token` containing `\|` is URL-encoded on `/2.3/login`; server accepts | Candidate* |
| ROLL-25 | `create_account` ignored | `/2.3/login` ignores `create_account` in body (account creation implicit on first PAP login) | Candidate* |
| ROLL-26 | Partial-PAP intermediate build | Version below `feature_203` min-version never enrolls; follows eero-auth until `feature_204` fires | Not Automated |

\* Depends on Kamino/AuthPortal OTP test-account access ([CORE-32890](https://eeroinc.atlassian.net/browse/CORE-32890)); manual until resolved.

## Open items to confirm before finalizing the plan

- Exact `feature_203` min-version (spec table says `26.14.0`; full client wiring stated to ship 26.12 — reconcile).
- PAP-complete release number (`X` in `papLoginRequiredCapable = isMobileParityCapable(26, X, 0)`).
- Whether org (`is_pro`) PAP login is in scope for the initial mobile cut (`CORE-32378 is_pro handling` is Backlog).
- Test-account strategy for AuthPortal OTP (blocks all `*`-marked cells).

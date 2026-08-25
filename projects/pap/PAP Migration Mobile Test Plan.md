# QA test plan

**PAP Migration Mobile Test Plan**

**Version**: 1.0

**Authors**: Henrique
**Template Version**: 3.1

*Note: Please read the test plan guidelines before starting to create your test plan*

## 1. Introduction

The PAP Migration moves eero's ~14M consumer "eero auth" users off eero's self-managed OTP authentication and onto an **Amazon Identity Private Account Pool (PAP)**, authenticated through **Amazon AuthPortal**. On mobile, the MAP SDK renders AuthPortal in a WebView; the client receives a **MAP token** and exchanges it with eero cloud at `POST /2.3/login` for an eero user session. The Welcome screen is restructured to a single "Amazon sign-in" plus an "Email or phone number" (PAP) entry, gated per device by `feature_203`. A separate `feature_204` forces pre-PAP clients to update before entering credentials. This plan covers the **mobile (iOS + Android)** surface only.

### 1.1 Links to Relevant Documentation

* Mobile Tech Spec: [PAP Migration - Mobile Tech Spec](https://docs.google.com/document/d/1yZ3M5ze90yTmJ3I15H0F2co84VP6EloHuWBLNR28zrg/edit) (local copy: [references/PAP-Migration-Mobile-Tech-Spec.md](./references/PAP-Migration-Mobile-Tech-Spec.md))
* Cloud HLD: [PAP Migration HLD](https://docs.google.com/document/d/1tllKzECjAGPHwXNzsTnGXMsy8FusrG1WC98eSnV4pis/edit)
* ERD: [PAP for eero Auth Users — ERD](https://docs.google.com/document/d/1mGoR0uoGjoKeqOD5w3BS-kvnyluwjPMsI-qC91rXLUQ/edit)
* JIRA Initiative: [INIT-351 — Private Account Pool (PAP)](https://eeroinc.atlassian.net/browse/INIT-351)
* Mobile Epic: [CORE-32410 — PAP Mobile Support](https://eeroinc.atlassian.net/browse/CORE-32410)
* Cloud/Passwordless Epic: [CORE-28017 — Support Passwordless login in PAP](https://eeroinc.atlassian.net/browse/CORE-28017)
* Figma: [SSO / Welcome screen](https://www.figma.com/design/WjOfX2phnwdS2Go0iPDNSY/SSO?node-id=738-1863)
* Testing Guide: [PAP ↔ Amazon Identity Testing Guide](https://eeroinc.atlassian.net/wiki/spaces/CLOUD/pages/5599264957/PAP+Amazon+Identity+Testing+Guide)
* Security Consult: [SEC-2470](https://eeroinc.atlassian.net/browse/SEC-2470)
* Slack: [#proj channel](https://eero.slack.com/archives/C0AA52DR28G)
* TestRail: TBD (section to be created once scenarios are approved)

### 1.2 Feature Flag

| Feature Flag | Renamed to | User Roles | Description |
| :---- | :---- | :---- | :---- |
| feature\_203 | `papLoginRequired` | All mobile clients (device-bucketed) | Enables the new Welcome screen (V2) + PAP flow; routes email/phone taps to AuthPortal → `/2.3/login`. Gains `X-Client-Device-Id` bucketing for `HASH_SHARD` percentage rollout. Min version 26.14.0 (table); full client wiring ships 26.12. |
| feature\_204 | `forceAppUpgradeToUsePap` | Pre-PAP clients (26.10 / 26.11) | Presents the "app update required" screen (`UpdateRequiredFragment` / `ErrorViewController`) before eero-auth credential entry. Min version moved 26.12.0 → 26.10.0. |

Server-side kill switches (Phase 4 cutover): `eeroAuthLoginDisabled` (→ `/2.2/login` + `/2.2/pro/login` 404) and `eeroAuthRegistrationDisabled` (→ `/2.2/register` 404). Error: `error.login.upgrade_required`.

### 1.3 eeroOS/App version

* **eeroOS**: All versions (auth is app/cloud-side; no gateway dependency)
* **iOS App**: 26.12 (feature_203 wiring); force-upgrade behavior validated on 26.10 / 26.11
* **Android App**: 26.12 (feature_203 wiring); force-upgrade behavior validated on 26.10 / 26.11

### 1.4 Milestone Breakdown

| Milestone | Scope | Story Points |
| :---- | :---- | :---- |
| Cloud — Passwordless + `/2.3/login` (CORE-28017) | PAP login route, backfill, credential sync, feature flags | In Execution |
| Android Implementation (CORE-32410) | WelcomeFragmentV2, MAP per-call config, `papLogin`, force-update, flag renames | Idea/Spike |
| iOS Implementation (CORE-32410) | Welcome V2 CTA region, EeroWebLoginCoordinator, `papLogin`, force-update, flag renames | Idea/Spike |

### 1.5 In Scope

- Welcome screen V2 rendering gated by `feature_203` (enrolled vs not; live toggle without relaunch on iOS)
- Three-entry routing: Technicians (SSO), Amazon sign-in (retail handle), Email or phone number (PAP handle `amzn_eero_mobile_us`)
- PAP sign-in via AuthPortal (email OTP) → MAP token → `POST /2.3/login` → eero session
- New user (`is_new_user=true`, marketing-consent fallback prompt) vs existing backfilled user (`pap_customer_id` resolves) via `/2.3/login`
- Retail Amazon sign-in preserved (`/2.2/login/amazon`, global MAP config unchanged)
- Force-app-upgrade screen for 26.10 / 26.11 when `feature_204` is on (before credential entry)
- Legacy-endpoint cutover behavior (404 + `error.login.upgrade_required`) and last-resort backstop for un-updated clients
- Feature-flag rollback behavior on next `app_configuration` refresh (feature_203 → 0%, feature_204 → off)
- Device-ID bucketing stability (`X-Client-Device-Id`: IDFV on iOS, app-scoped UUID on Android)
- Duplicate-account detection (PAP account whose email/phone belongs to an existing Amazon-login user)
- `/2.3/login` contract (body `auth_token`, `is_pro`; header `X-Client-Device-Id`; `create_account` ignored; `|` URL-encoding)
- Account-linking footnote / entry on Welcome V2
- Analytics: `LoginAnalyticsEvents.StartedLogin` per auth type
- Platform parity: iOS and Android

### 1.6 Out of Scope

- Insight and account.eero.com (web surfaces — CORE-32858; separate plan)
- Cloud-internal correctness of backfill / reconciliation APIs and CES consumer (unit/integration owned by Cloud; verified server-side, not via mobile)
- Amazon-login users and SSO users (unaffected by PAP)
- Password / passkey support (not at launch)
- Third-party login (Google/Apple)
- eero Business (EB) users
- Org / `is_pro` PAP login **if** CORE-32378 (is_pro handling on `/2.3/login`) is not in the initial mobile cut (confirm — see §7)
- Unit tests (SDE-owned; not in QA scope)

### 1.7 Sign off Criteria

- All P0/P1 scenarios pass on both iOS and Android
- No open Sev1 or Sev2 defects
- `feature_203` correctly gates Welcome V2 and the PAP flow; unenrolled/below-min-version clients stay on legacy path
- `feature_204` shows the update-required screen on 26.10/26.11 and never blocks 26.12+ PAP users
- `/2.3/login` returns a valid session for new and existing users; `is_new_user` drives the consent prompt correctly
- Retail Amazon sign-in unaffected throughout
- Rollback (both flags) restores prior behavior on next `app_configuration` refresh with no stranded users
- Platform parity confirmed (iOS/Android)
- Regression suite passes with no new failures
- Accessibility audit passed (VoiceOver + TalkBack)

---

## 2. Test Environment

### 2.1 Hardware Models and eeroOS version

| Hardware | eeroOS Version |
| :---- | :---- |
| Any eero gateway | Any (no gateway dependency for auth) |

### 2.2 Mobile Devices and OS version

| Platform | Device | OS Version |
| :---- | :---- | :---- |
| iOS | iPhone 14, iPhone 15 Pro, iPhone SE 3rd gen | iOS 16, iOS 17, iOS 18 |
| Android | Pixel 7, Samsung Galaxy S23, Samsung Galaxy A54 | Android 13, Android 14, Android 15 |

App-version bands under test: **26.10 / 26.11** (pre-PAP, force-upgrade path) and **26.12+** (PAP-complete).

### 2.3 Environment / Backend

| Environment | AuthPortal host | Assoc handle | OTP / auth | Notes |
| :---- | :---- | :---- | :---- | :---- |
| **Devo** | `development.amazon.com` flow | `amzn_eero_mobile_us` (PAP) | Hardcoded OTP `112233`; web `at-*` token mintable | `/2.3/login` confirmed working; debug admin routes provision accounts + `users` row (CORE-31962) |
| **Stage** | `ap.stage.payments.e2ro.com` | `amzn_eero_mobile_dogfood_us`, marketplace `A10ZONQX51YR1E` | Existing stage PAP account with password `123123` (email+password, no OTP) | More deterministic for automation |
| **Prod** | `ap.account.eero.com` | `amzn_eero_mobile_us` | AuthPortal OTP (no headless retrieval) | Kamino + APEX for programmatic MAP tokens — **pending validation** (CORE-32890) |

### 2.4 Network Configuration

- Standard residential network with owner account
- Network with an org / Pro account (for `is_pro` path, if in scope)

---

## 3. Test Data

### 3.1 User type

| User Type | Account state | feature\_203 | feature\_204 | App version | Expected behavior |
| :---- | :---- | :---- | :---- | :---- | :---- |
| Existing eero-auth (backfilled) | `pap_customer_id` set, `pap_linked_at` set | Enrolled | Off | 26.12+ | PAP sign-in resolves to existing `users` row; `is_new_user=false` |
| New user | No prior eero account | Enrolled | Off | 26.12+ | `/2.3/login` creates `users` row; `is_new_user=true`; consent prompt |
| Enrolled, retail | Amazon-login user | Enrolled | Off | 26.12+ | Amazon sign-in unchanged (`/2.2/login/amazon`) |
| Not enrolled | eero-auth | Not enrolled | Off | 26.12+ | Legacy Welcome V1 + eero-auth |
| Pre-PAP client | eero-auth | Invisible (below min) | Off | 26.10 / 26.11 | Legacy path; no update prompt |
| Pre-PAP client at cutover | eero-auth | n/a | On | 26.10 / 26.11 | Update-required screen before credential entry |
| Org / Pro (`is_pro=true`) | Org member | Enrolled | Off | 26.12+ | Org membership + IP allowlist; Insight-enabled session (if in scope) |
| Duplicate | email/phone matches existing Amazon-login user | Enrolled | Off | 26.12+ | New PAP account deleted; `error.form.email/phone.unavailable` |

### 3.2 Endpoint / response type

| Case | `/2.3/login` response | Meaning |
| :---- | :---- | :---- |
| Existing user | `{ data: { user_token, is_new_user: false } }` | Session for backfilled user |
| New user | `{ data: { user_token, is_new_user: true } }` | New `users` row + `pap_linked_at`; consent fallback prompt |
| Invalid/expired token | 4XX | Aztec token validation failure |
| Legacy cutover | `/2.2/login*` / `/2.2/register*` → 404 | `error.login.upgrade_required` |

### 3.3 Test-account tooling (see [test-user-login-concerns.md](./test-user-login-concerns.md))

- **Devo:** debug admin routes (`/debug/pap/create_account` with `create_user=true`, `set_email`, `verify_token`, `is_authorized`, `get_contact_info`); OTP `112233`; web `at-*` token accepted by Aztec `IsAuthorized`. Constraint: `set_mobile` blocked (email-only/EOA for now); `get_contact_info` UCI fields empty pending RED-cert.
- **Stage:** existing PAP account, password `123123`.
- **Prod/Stage MAP tokens:** Kamino + APEX device registration — **blocker pending** (CORE-32890).
- **Mobile UI:** nova-act on Device Farm drives the real AuthPortal WebView (not headless).

---

## 4. Test Strategy

### 4.1 Manual Scope

- **Exploratory** testing of Welcome V2, PAP sign-in (email), Amazon sign-in, Technicians routing on both platforms
- **Flag-state matrix** validation (enrolled/not, feature_204 on/off) across app-version bands
- **Force-upgrade** flow on 26.10 / 26.11 (update-required screen, App Store button, retail sign-in not blocked)
- **Legacy cutover** behavior (404 backstop) and **rollback** behavior (next `app_configuration` refresh)
- **Duplicate-account** detection and error messaging
- **Consent** fallback prompt for new users (`is_new_user=true`)
- **Analytics** validation (Charles Proxy): `StartedLogin` per auth type
- **Accessibility** (VoiceOver / TalkBack) across Welcome V2 and AuthPortal WebView
- **Visual** validation against SSO Figma

### 4.2 Automated scope

#### 4.2.1 SDE
- Unit tests for Welcome routing on `feature_203`; flag accessor renames; MAP per-call param wiring. (SDE-owned; listed for awareness, not QA scope.)

#### 4.2.2 QAE — **when and if feasible** (gated by test-account strategy)
- **API (`/2.3/login`)** — feasible now in **devo** (mint web `at-*` token via AuthPortal + OTP `112233`) and **stage** (password `123123`). Covers new vs existing user, token validation, contract. See kickoff in api-tests room.
- **Mobile UI** — nova-act / Device Farm through AuthPortal; blocked on OTP-in-UI handling (Q-QA-5) for stage/prod.
- **Prod/stage MAP-token headless path** — blocked on Kamino + APEX validation (Q-QA-1 / CORE-32890).
- RC builds validate; nightly builds for automation only.
- Validate whether mock endpoints are needed for cutover/404 and rollback simulation.

### 4.3 Regression

- Existing eero-auth login/registration (valid during transition only; retired at cutover)
- Retail Amazon-login flow (`/2.2/login/amazon`) — must remain unaffected
- Onboarding routing (`V3OnboardingActivity` on Android; `WelcomeCoordinator` on iOS)
- Existing login helper coverage (`test_login_utils`) — will need replacement for PAP UI login

---

## 5. Test Tools

| Tool | Purpose |
| :---- | :---- |
| TestRail | Test case management (section TBD) |
| Jira | Defect management and tracking |
| Charles Proxy | Analytics + `/2.3/login` request/response inspection; `X-Client-Device-Id` header |
| Xcode / Android Studio | Debug builds, feature-flag toggling (debug menu overrides) |
| Appium / nova-act (Device Farm) | E2E mobile automation through AuthPortal (when feasible) |
| Kamino / APEX | Programmatic MAP-token test accounts (pending validation) |
| Debug admin routes (`/debug/pap/*`) | Devo account + `users` row provisioning; token minting |
| Accessibility Inspector / TalkBack | Accessibility validation |

## 6. Test Scenarios for eero App

Each scenario is one complete user flow. IDs map to the rollout-state matrix in [test-scenarios-rollout.md](./test-scenarios-rollout.md) (ROLL-\*). `*` = automation depends on AuthPortal OTP test-account access (CORE-32890); manual/UI-only until resolved.

| \# | TestRail | ID | Requirements | Test Scenario | Automation |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Welcome Screen V2 & Routing** | | | | | |
| 1 | TBD | WEL-01 | V2 renders when enrolled | Login-capable 26.12+ device, `feature_203` enrolled → Open app → Verify Welcome V2 (Technicians pill, "Amazon sign-in", "Email or phone number", account-linking footnote) | Candidate |
| 2 | TBD | WEL-02 | V1 renders when not enrolled | 26.12+ device NOT enrolled → Open app → Verify legacy Welcome V1 ("Get started" / "Sign in"); V2 never shown | Candidate |
| 3 | TBD | WEL-03 | Flag invisible below min-version | 26.10/26.11 device → Open app → Verify legacy Welcome V1 (feature_203 not received) | Candidate |
| 4 | TBD | WEL-04 | Live toggle without relaunch (iOS) | iOS enrolled → Toggle `feature_203` via debug menu → Verify Welcome layout + Technicians toolbar react without relaunch | Not Automated |
| 5 | TBD | WEL-05 | Technicians routing | Welcome V2 → Tap Technicians → Verify SSO flow (`SsoOtherPartnerFragment` / `SSOCoordinator`) | Candidate |
| 6 | TBD | WEL-06 | Amazon sign-in routing | Welcome V2 → Tap "Amazon sign-in" → Verify retail Amazon-login (handle `amzn_eero_mobile_android_us`), global MAP config unchanged, hits `/2.2/login/amazon` | Candidate |
| 7 | TBD | WEL-07 | Email/phone routes to PAP | Welcome V2 → Tap "Email or phone number" → Verify AuthPortal WebView opens (host `ap.account.eero.com`, handle `amzn_eero_mobile_us`) | Candidate\* |
| **PAP Sign-In (`/2.3/login`)** | | | | | |
| 8 | TBD | SIGN-01 | New user, email OTP (happy path) | Enrolled 26.12+ → Email or phone → Enter email → AuthPortal OTP → Verify MAP token exchanged at `/2.3/login` → `is_new_user=true` → `users` row + `pap_linked_at` created → session; marketing-consent fallback prompt shown | Candidate\* |
| 9 | TBD | SIGN-02 | Existing (backfilled) user | Enrolled 26.12+, user with `pap_customer_id` set → PAP sign-in → Verify resolves to existing `users` row → `is_new_user=false` → verified session | Candidate\* |
| 10 | TBD | SIGN-03 | Phone claim | Enrolled 26.12+ → PAP sign-in with phone number → AuthPortal phone OTP → Verify session (note devo `set_mobile` constraint — validate on stage/prod) | Candidate\* |
| 11 | TBD | SIGN-04 | Invalid/expired token | Force invalid/expired MAP token → `/2.3/login` → Verify 4XX handled gracefully in UI (Aztec validation failure) | Candidate\* |
| 12 | TBD | SIGN-05 | Org / Pro (`is_pro=true`) | Enrolled 26.12+ org user → PAP sign-in with `is_pro=true` → Verify org membership + IP allowlist enforced; Insight-enabled session (**if in scope**) | Candidate\* |
| 13 | TBD | SIGN-06 | Contract — encoding & header | Inspect `/2.3/login` request → Verify `auth_token` `\|` URL-encoded, `X-Client-Device-Id` present (IDFV/UUID), `create_account` ignored | Candidate\* |
| **Duplicate Account & Linking** | | | | | |
| 14 | TBD | DUP-01 | Duplicate detection | PAP sign-in where resolved email/phone belongs to an existing Amazon-login user → Verify new PAP account deleted (`DeleteAccountV2`) and `error.form.email.unavailable` / `error.form.phone.unavailable` | Candidate\* |
| 15 | TBD | LINK-01 | Account-linking footnote | Welcome V2 → Verify account-linking footnote copy below buttons; tap-through behaves per Figma | Not Automated |
| **Force Upgrade (feature_204)** | | | | | |
| 16 | TBD | UPG-01 | Update-required on pre-PAP | 26.10/26.11, `feature_204` on → Tap email/phone sign-in → Verify update-required screen (`UpdateRequiredFragment` / `ErrorViewController`) shown before credential entry, with App Store button | Candidate |
| 17 | TBD | UPG-02 | Retail sign-in not blocked | 26.10/26.11, `feature_204` on → Tap Amazon sign-in → Verify retail login still reaches `/2.2/login/amazon` (not blocked) | Candidate |
| 18 | TBD | UPG-03 | 26.12+ never blocked | 26.12+ enrolled, `feature_204` on → PAP sign-in → Verify update check never reached; PAP flow proceeds | Candidate\* |
| **Legacy Cutover (Phase 4)** | | | | | |
| 19 | TBD | CUT-01 | Legacy routes 404 | `eeroAuthLoginDisabled` + `eeroAuthRegistrationDisabled` flipped → Call `/2.2/login`, `/2.2/pro/login`, `/2.2/register` → Verify 404 + `error.login.upgrade_required` | Candidate |
| 20 | TBD | CUT-02 | Un-updated client backstop | 26.10/26.11 user who ignored the update prompt after cutover → Attempt eero-auth → Verify raw 404 backstop | Candidate |
| 21 | TBD | CUT-03 | PAP unaffected at cutover | 26.12+ PAP user after cutover → Sign in → Verify `/2.3/login` still 200 | Candidate\* |
| **Rollback** | | | | | |
| 22 | TBD | RBK-01 | feature_203 → 0% | Enrolled 26.12+ → Drop `feature_203` throttle to 0% → Refresh `app_configuration` → Verify fallback to Welcome V1 + eero-auth; legacy endpoints still serve; no stranded user | Candidate |
| 23 | TBD | RBK-02 | feature_204 → off | 26.10/26.11 seeing update prompt → Disable `feature_204` → Refresh `app_configuration` → Verify prompt no longer shown | Candidate |
| **Device Bucketing** | | | | | |
| 24 | TBD | BKT-01 | Enrollment stability | Same `X-Client-Device-Id` across multiple sessions → Verify consistent enrolled/not-enrolled (`HASH_SHARD`); reinstall (IDFV reset) may re-bucket — document behavior | Not Automated |
| **Analytics** | | | | | |
| 25 | TBD | ANA-01 | StartedLogin per type | Welcome V2 → Tap each entry → Verify `LoginAnalyticsEvents.StartedLogin` fires with correct auth type (`amazonSignIn` / `emailOrPhoneSignIn` / `technicianSignIn`) | Not Automated |
| **Edge Cases** | | | | | |
| 26 | TBD | EDGE-01 | Network error during AuthPortal | PAP sign-in → Enable airplane mode mid-flow → Verify graceful error → Recover → Retry succeeds | Not Automated |
| 27 | TBD | EDGE-02 | App backgrounded during sign-in | Start PAP sign-in → Background app in AuthPortal WebView → Return → Verify flow state preserved or recoverable | Not Automated |
| 28 | TBD | EDGE-03 | Partial-PAP intermediate build | Build below `feature_203` min-version → Verify never enrolls; follows eero-auth until `feature_204` fires | Not Automated |
| **Accessibility** | | | | | |
| 29 | TBD | A11Y-01 | VoiceOver/TalkBack Welcome V2 + AuthPortal | Enable VoiceOver (iOS)/TalkBack (Android) → Navigate Welcome V2 → Verify entries announced/labeled → Enter AuthPortal → Verify WebView fields announced | Not Automated |

---

## 7. Risks

| \# | Risk | Impact | Likelihood | Mitigation |
| :---- | :---- | :---- | :---- | :---- |
| 1 | **Kamino + APEX MAP-token path unvalidated for eero PAP marketplace** | Prod/stage PAP login automation blocked; PAP flows manual/UI-only for launch | High | Track CORE-32890 (Triage, unassigned — needs owner); devo + stage-password paths viable in the interim; PoC in Phase 2 |
| 2 | No headless OTP under PAP (by design) | Current headless login helper (`test_login_utils`) invalid for PAP UI login | High | nova-act/Device Farm through AuthPortal; devo OTP `112233`; stage password `123123` |
| 3 | `feature_203` min-version ambiguity (26.14.0 table vs "ships 26.12") | Wrong version band tested; enrollment assumptions off | Medium | Reconcile with Cloud/Mobile before execution (see Open Items) |
| 4 | Flag race conditions mid-session (iOS shared state) | Partial/incorrect Welcome render | Low | WEL-04 live-toggle test on both platforms |
| 5 | Retail Amazon-login regression from shared MAP config | Retail users unable to sign in | Medium | WEL-06 / UPG-02 explicitly assert `/2.2/login/amazon` unaffected; global MAP config untouched |
| 6 | Cutover 404 stranding un-updated clients | Users locked out with poor error | Medium | UPG-01 forces update first; CUT-02 backstop; rollback (RBK) is two-way door |
| 7 | Devo `set_mobile` blocked / UCI fields empty | Phone-claim + contact-info scenarios not testable in devo | Medium | Validate phone path on stage/prod; track RED-cert/UCI allow-list |
| 8 | Duplicate-account detection edge cases | Wrong error or orphaned PAP account | Medium | DUP-01; monitor `DeleteAccountV2` outcomes |
| 9 | Org / `is_pro` handling on `/2.3/login` is Backlog (CORE-32378) | SIGN-05 may be out of scope at launch | Medium | Confirm scope before execution; mark SIGN-05 conditional |
| 10 | OS/version fragmentation across 26.10 / 26.11 / 26.12+ | Behavior differences across bands | Medium | Test min/max supported OS per platform and each app-version band |
| 11 | Device re-bucketing on reinstall (IDFV/UUID reset) | Test user changes enrollment unexpectedly | Low | BKT-01 documents behavior; pin device IDs where possible |

---

## 8. Sign-Off

| Team | DRI (person signing off) | Approval Status |
| :---- | :---- | :---- |
| QA | Henrique |  |
| QA | Internal QA Reviewer |  |
| Cloud / Auth | Burak Varan |  |
| Mobile | Adauton Heringer |  |
| SDM | SDM or Project Lead |  |
| Product | Product Manager |  |

**Note:** "Informed sign off" means the DRI from the team (or someone to whom they have delegated) should sign off to acknowledge that they have read the document and had the opportunity to share their questions and concerns.

## Open Items to Confirm Before Execution

- Exact `feature_203` min-version (26.14.0 table vs 26.12 wiring) and the PAP-complete release number `X` in `papLoginRequiredCapable`.
- Whether org (`is_pro`) PAP login (CORE-32378, Backlog) is in the initial mobile cut → gates SIGN-05.
- Owner + lead time for Kamino/APEX test-account setup (CORE-32890 unassigned) → gates all `*` automation.
- Which environment RC builds validate against (devo vs stage) and whether existing automation users migrate to the PAP.

---

*End of Test Plan*

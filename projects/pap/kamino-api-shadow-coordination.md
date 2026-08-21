# PAP — Kamino + API Shadow Testing: Cloud Coordination Plan

Working plan for QA to orchestrate the test-account / API shadow-testing effort with the cloud team. Anchored on [CORE-32890 — Investigate Kamino test accounts](https://eeroinc.atlassian.net/browse/CORE-32890) (currently Triage, unassigned).

## Goal

Prove we can log test users into `POST /2.3/login` **headlessly** (mint a MAP token for an eero-PAP test account, call the API), then shadow-test the full cloud contract at the API level and produce an expected-app-behavior matrix — all before the app UI or Device Farm automation exists. See [test-user-login-concerns.md](./test-user-login-concerns.md) for the underlying blocker.

## The unlock

Everything below is gated on producing **one** valid MAP token for a Kamino eero-PAP account. Candidate routes, in order of "ask cloud first":

1. **Cloud debug routes (CORE-31962 — Done):** the "debug admin routes to test Identity clients" may already return a MAP token or a usable login — potentially removing the need for APEX entirely. **Confirm first.**
2. **Kamino + APEX device registration:** the ADCMS/Alexa pattern — but unproven against a non-retail PAP marketplace.
3. **Devo hardcoded OTP `112233`:** scriptable in devo if a registration sequence can be driven with a fixed code.

## API automation status (Aug 21, 2026)

Harness scaffolded in the api-tests room (thread "PAP /2.3/login API automation"). Decisions:
- **Native `fetch`**, not the `eero api` CLI wrapper — matches repo convention and sidesteps the `+ / = |` quoting gotcha.
- **Schemas match the guide:** error identifier is `meta.message` (not a top-level field); debug-route `meta` omits `server_time`.
- **Playwright loaded on demand** (deliberately not in package.json/lockfile — adding it without a CodeArtifact-resolved entry breaks `npm ci`); helper errors with an install hint if missing.
- `mint_at_token` supports password sign-in (stage account, `123123`) and OTP `112233` (devo forgot-password).

**Runnable today (deterministic):** happy path, unknown-CID (`error.login.unknown`), malformed token (`external_token_invalid`).
**Gated / TODO (not faked):** 503 Aztec-outage (needs fault injection); `is_pro=true` 403 paths (need eph redeploy on `71859e3`).

**STAGE-first (confirmed by Burak, Aug 21):** the happy path targets **stage**, which is simpler than the guide implied —
- Base host `api-user.stage.e2ro.com` → `POST https://api-user.stage.e2ro.com/2.3/login`.
- **No eph header** (persistent stage, not an eph).
- **No admin token / no create_account for the happy path** — an existing stage PAP account is already provisioned (has its users row). Password `123123`; email is in Ricardi's #proj-pap-eero-login recording (to be supplied via env).
- `mint_at_token` uses **password sign-in** at the stage AuthPortal (`ap.stage.payments.e2ro.com`, assoc handle `amzn_eero_mobile_dogfood_us`, marketplace `A10ZONQX51YR1E`) — deterministic, no OTP.
- Henrique's `ADMIN_TOKEN` in `.zshrc` is a **stage** token (63 chars) — only needed for the provisioning-dependent scenarios via `/debug/pap/*`, not the happy path.
- Env vars: `PAP_STAGE_BASE_URL=https://api-user.stage.e2ro.com`, `PAP_STAGE_EMAIL=<from recording>`, `PAP_STAGE_PASSWORD=123123`.

**Only remaining value for the first green run:** the stage account email (from Ricardi's recording). Devo path (OTP 112233 + admin token + create_account) stays as a secondary parameterized config for unknown-CID etc., pending devo creds.

**Open questions raised to cloud/Identity** (see message sent to #proj-pap-eero-login): devo creds/hosts; non-browser token mint (or accept headless Playwright for CI); stage account email + is `123123` CI-safe; `71859e3` redeploy timeline; rate limits on `create_account`/AuthPortal; `set_mobile` unblock timeline (MOA/MCA); OK to prove `external_login_failed` by presenting a devo token to stage (only deterministic wrong-pool repro).

## Division of labor (proposed)

| Area | Owner |
|------|-------|
| Confirm eero PAP marketplace config (association handle `amzn_eero_mobile_us`, marketplace ID, AuthPortal domain, which envs) | Cloud |
| Confirm/enable Kamino account creation in the eero PAP (devo + stage) | Cloud + Identity |
| Prove token minting (debug route or APEX) for a PAP account | Cloud-led, QA pairs |
| Stage/devo `/2.3/login` endpoint + any allowlisting for the test harness | Cloud |
| Define expected `/2.3/login` responses per branch | Cloud + QA |
| Build pytest API harness (mint token → call `/2.3/login` → assert) | QA |
| Scenario matrix + error injection + expected-app-behavior mapping | QA |
| Feed results to the later Device Farm / nova-act UI suite | QA |

## Asks for the cloud team

1. eero PAP marketplace details for test-account creation: association handle, marketplace ID, AuthPortal domain, and which environments the PAP exists in (devo/stage/prod).
2. Can Kamino create accounts in the eero PAP today? Do the CORE-31962 debug admin routes already give us a MAP token or a usable login (so we may not need APEX)?
3. Who can pair on validating APEX device registration against our PAP marketplace — the open PoC in CORE-32890, currently unassigned. Assign an owner.
4. A stage/devo `/2.3/login` endpoint QA can hit, plus any allowlisting needed to call it from a harness.
5. Ability to pre-provision or self-create accounts in these states: backfilled/mapped, brand-new AuthPortal user, and one whose email/phone collides with an existing Amazon-login user (for duplicate detection).
6. Whether error branches (Aztec failure, backfill gap = CID with no `users` row, kill-switch 404) arise naturally from data setup, or need a debug hook to simulate.

## Sequence / milestones

1. **Kickoff (30 min):** confirm marketplace + env; confirm whether debug routes already unblock a token; assign CORE-32890.
2. **Token PoC:** one MAP token for a Kamino eero-PAP account → successful `/2.3/login`. (Blocker for everything after.)
3. **QA harness:** pytest, token-in → `/2.3/login` → capture response.
4. **Shadow suite:** run across account types (EOA/MOA/MCA, mapped/new/colliding) + error injections.
5. **Behavior matrix:** response → predicted app behavior; hand to UI automation as the oracle.

## Scenario matrix (API shadow → predicted app behavior)

| Drive `/2.3/login` with… | Cloud returns | Predicted app behavior |
|--------------------------|---------------|------------------------|
| Kamino account already mapped (`pap_customer_id` set) | `200`, `user_token`, `is_new_user=false` | Signs in, proceeds to network/home |
| Kamino account with no eero row (new AuthPortal user) | `200`, `is_new_user=true` | Session created + marketing-consent fallback prompt |
| PAP account colliding with existing Amazon-login user | `error.form.email.unavailable` / `phone.unavailable` + `DeleteAccountV2` | "Already in use" error |
| `is_pro=true`, valid org member + allowed IP | Org/Insight session | App enters org context |
| `is_pro=true`, non-member / disallowed IP | Authz failure | App blocks org login |
| Expired / revoked / wrong-marketplace / malformed token | Aztec validation failure | Generic login error |
| Valid CID, AddressService can't fetch contact info | Lookup failure | "User-not-found after auth" error |
| Old endpoint after kill switch | `404 error.login.upgrade_required` | Force-update / generic failure |

Bottom rows (token + backfill-gap failures) are **easier to force at the API than in the UI** — a reason to lead with the API path.

## Out of scope for the API path (needs Device Farm / UI)

AuthPortal WebView rendering, eero branding/skinning/localization, device registration + OTP entry UX, Welcome V2 routing (`feature_203`), force-update screen (`feature_204`), actual on-device rendering of each response.

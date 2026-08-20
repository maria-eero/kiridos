# PAP — Test-User Login Concerns (QA Blocker Analysis)

> This is the reason the test plan is on hold. It captures why authenticating test users is fundamentally different under PAP than under today's eero auth, what the docs say the intended approach is, and the open questions that must be answered before we can commit an automation strategy.

## TL;DR

Under eero auth today, QA automation gets test users logged in by **retrieving the OTP via an admin API** and submitting it — fully headless. **Under PAP this no longer works.** Post-migration, login goes through Amazon AuthPortal and returns a **MAP token**, which the client exchanges at `POST /2.3/login`. There is **no headless OTP retrieval in prod/stage** (by design, for security). Getting a test user "logged in" now means obtaining a valid MAP token, and the only documented programmatic path for that in prod/stage is **Kamino + APEX device registration** — which is **not yet validated against the eero PAP marketplace**.

## What changes vs today

| Aspect | Today (eero auth) | Post-migration (PAP) |
|--------|-------------------|----------------------|
| Login mechanism | eero user-service OTP; `POST /2.2/login` + `/2.2/login/verify` | AuthPortal (MAP SDK web view) → MAP token → `POST /2.3/login` |
| Headless OTP retrieval (automation) | Available via admin API (used by current mobile suite) | **Not available in prod/stage** (security) |
| Token needed to reach the API | eero session token from OTP verify | MAP token from Amazon Identity |
| Test-account source | eero test users (e.g. cloud_smoke, network_admin) with email/phone verified | PAP test accounts that can vend MAP tokens |

The existing automation test-user matrix (cloud_smoke / cloud_smoke2 / network_admin / sofy_brain / mobile_smoke / die_hard, etc.) relies on the admin-API OTP path. That path is **irrelevant to PAP login** — those users, as configured today, cannot authenticate through the PAP/MAP flow in automation without a MAP-token strategy.

## What the docs prescribe (HLD "Testing Considerations")

Post-migration, headless login is not available through the PAP (no OTP retrieval in prod/stage for security reasons).

- **Devo:** accounts created through `development.amazon.com` use hardcoded OTP `112233`.
- **Prod/Stage:** create test accounts via **Kamino + APEX device registration** to obtain MAP tokens programmatically, then call `/2.3/login` with the MAP token. Same pattern used by **ADCMS / Alexa** for MAP-secured API testing. **Requires validation that the flow works against the eero PAP marketplace.**
- **Mobile automation (nova-act on Device Farm):** must go through the AuthPortal flow (UI, not headless).
- **Integration tests:** existing eero-auth user-service tests stay valid during transition; new tests needed for PAP login, credential sync, and backfill.

### Key Decision #5 (HLD): Kamino over Tipoca

> Kamino accounts can produce MAP tokens via APEX device registration (required for `/2.3/login`). Tipoca only vends auth tokens / at-main cookies (MAP tokens out of scope). **Pending validation that Kamino + APEX works against our PAP marketplace.**

## Test-account tooling comparison (from AbeBooks AuthPortal Integration Testing Spike + TipocaService wiki)

| Tool | What it vends | Devo? | Prod? | UI? | Fit for PAP MAP-token testing |
|------|---------------|-------|-------|-----|-------------------------------|
| **Kamino** | Devo (non-expiring) + Prod (expiring) test accounts; calls Tipoca in backend; bypasses CAPTCHA/many security challenges. `KaminoServiceJavaClient` for automation. Tier-2, onboarding SLA ~3 days (prod issues 5d, devo 10d). | Yes | Yes | Yes | **Chosen.** MAP tokens via APEX device registration. Precedent: ShopBop, IMDB use Kamino for LWA/auth testing. |
| **Tipoca** | Basic email-claim prod accounts; supports "PAP Tipoca"; 10,000 accounts for load/gameday. | **No** | Yes | No (code via Brazil/AAA) | Rejected — MAP tokens out of scope; only auth tokens / at-main cookies; no devo. |
| AuthService (direct) | Prod accounts | No | Yes | — | Devo doesn't support test accounts. |
| KILO | Large-scale gameday accounts | — | — | — | Not our use case. |

**Precedent caveat (AbeBooks spike open question):** for PAP subsidiaries there did **not** appear to be a straightforward path to create **devo** accounts in a private account pool (no subsidiary locale option) — this exact gap is unresolved and mirrors our Q below.

## Open questions to resolve before the test plan

1. **Does Kamino + APEX device registration actually produce valid MAP tokens for the eero PAP marketplace?** This is an explicit "pending validation" in the HLD and the crux of the whole automation approach. Tracked by **[CORE-32890 — Investigate Kamino test accounts](https://eeroinc.atlassian.net/browse/CORE-32890)** (currently **Triage, unassigned**) and Phase-2 PoC "Validate Kamino + APEX device registration produces MAP tokens for eero PAP marketplace — **blocker for stage automation**."
2. **Can we create devo PAP test accounts at all?** Devo uses hardcoded OTP `112233`, but the AbeBooks precedent flagged that devo account creation in a PAP (no subsidiary locale option) was itself an open question. Confirm eero's devo/dogfood story.
3. **What is the QA environment story — devo vs stage vs prod?** Devo has hardcoded OTP; stage/prod need Kamino. Which environment will RC builds validate against, and which do current test users live in?
4. **Do our existing automation test users migrate to the PAP, and can they then vend MAP tokens?** Or must we provision brand-new Kamino-backed PAP test accounts? What happens to the current cloud_smoke / network_admin / mobile_smoke / die_hard matrix?
5. **Mobile UI automation:** nova-act on Device Farm must drive the real AuthPortal web view. How do we handle OTP entry in a UI test when there's no headless retrieval — does Kamino/APEX bypass the OTP challenge, or is a code-injection/hardcoded-OTP path available in the test environment?
6. **`is_pro` / org login and Frontier/partner users:** how are org and partner-created test users authenticated for testing, given `/2.2/pro/login` (headless) has no PAP equivalent?
7. **Who owns setting up test accounts, and what is the onboarding lead time?** Kamino onboarding SLA is ~3 days but PAP-subsidiary onboarding is unproven; CORE-32890 is unassigned.

## Impact on the test plan (once unblocked)

- Automation feasibility for PAP login flows hinges entirely on Q1. If Kamino + APEX validates, automated `/2.3/login` coverage is viable (MAP token → user token). If not, PAP login is **manual / UI-only** for launch.
- Regardless, **mobile UI login through AuthPortal** likely needs nova-act/Device Farm coverage, not the current headless helper (`test_login_utils`).
- Existing eero-auth login coverage stays valid **during transition only** (Phase 2) and must be retired at cutover (Phase 3/4).
- New test surfaces to plan: duplicate-account detection, credential sync (eero↔PAP), backfill/reconciliation correctness, force-update screens (feature_204), Welcome screen V2 routing (feature_203).

## Sources

- PAP Migration HLD — "Testing Considerations", "Key Decisions for Review" #5, "Required Work / Phase 2".
- PAP Migration - Mobile Tech Spec — `/2.3/login` contract, MAP config, feature flags.
- AbeBooks Integration Test Spike (AuthPortal Integration Testing Spike) — Kamino vs Tipoca analysis.
- TipocaService wiki — test-account tooling comparison table.

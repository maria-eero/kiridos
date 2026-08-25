# Product Requirements

## User-Facing Requirements

1. **Registration** — new users create an eero account using email + optional phone, with OTP verification.
2. **Login** — existing users sign in with email or phone + OTP verification.
3. **eero branding** — auth experience is eero-branded (logo, colors, legal/copyright, CS links) via AUI skinning of AuthPortal.
4. **Locale support** — must match eero's current locale coverage. AuthPortal does **not** currently support Irish and Icelandic (~6,000 users) — gap must be closed via Panther locale onboarding through AuthX before launch.
5. **Consent collection** — ToS (locale-specific, supported) and marketing email consent collected during registration. Marketing consent needs an extra optional checkbox (Identity work) OR opt-in post-registration in account settings.
6. **Account maintenance** — users update email/phone. Post-migration this opens an AuthPortal web view; CES notifies cloud of changes.

## System Requirements

7. **Credential migration** — all eero auth users' email/phone/name migrated to the PAP as passwordless accounts (EOA/MOA/MCA).
8. **Identity mapping** — each migrated user's PAP CID stored in `users.pap_customer_id`.
9. **Account linking** — PAP users can link their Amazon retail account (matching current behavior); after linking, the PAP account is deactivated/deleted.
10. **Session creation** — after AuthPortal auth, cloud validates the token and creates an eero session so existing APIs keep working.
11. **Organization login** — non-SSO org users authenticate through AuthPortal, get a CID; org association, IP allowlist, and rate limiting stay in eero cloud (via `is_pro`).
12. **Partner user creation** — external partners (e.g., Frontier) that create verified eero users via API must keep working; each partner-created user also gets a PAP account. **Migration is therefore never fully complete** (ongoing partner pipeline).
13. **Throttle-based cutover** — a feature flag/throttle controls native eero-auth vs AuthPortal per client. Decision happens **before** the user enters credentials, so user-targeted rollout (by phone/email/network) is **not feasible** — rollout is percentage-based and per-client.
14. **Credential source of truth** — post-migration the PAP owns credentials; eero code reading email/phone from `users` must read from Identity (AddressService) or stay synced via CES.

## Client Requirements

15. **Mobile (iOS, Android)** — integrate MAP SDK for AuthPortal auth; minimum app version required. Android should migrate the deprecated `.jar` MAP SDK to the Brazil-based system (consider in scope).
16. **Insight** — AuthPortal web view; server-side, no version dependency.
17. **account.eero.com** — AuthPortal web view; must preserve the OAuth authorization flow and Frontier cobranding.
18. **Partner APIs** — login/verify flow formally deprecated; partners directed to machine tokens; existing user tokens keep working.

## Mobile UX — Welcome Screen (new)

> Figma screen exports for the full PAP sign-in / create-account / OTP-verify flow are cataloged in **[screens.md](./screens.md)** (`figma/`).

The Welcome screen is restructured to three top-level entries (replacing the two-button layout + bottom-sheet pickers):

| Button | Label | Routes to | Association handle |
|--------|-------|-----------|--------------------|
| Top-left pill | Technicians | Existing SSO flow | N/A (external partner SSO) |
| Primary | Amazon sign-in | Existing retail Amazon-login | `amzn_eero_mobile_android_us` |
| Secondary | Email or phone number | **New PAP flow** | `amzn_eero_mobile_us` |

- **Android:** new `WelcomeFragmentV2` alongside `WelcomeFragment`; routed by `V3OnboardingActivity.checkExtrasAndRoute()` on `feature_203`. Interactions: Technicians → `SsoOtherPartnerFragment`; Amazon sign-in → `SignInWithAmazonFragment` (existing retail handle); Email or phone → `SignInWithEeroWebFragment` (`amzn_eero_mobile_us`). Old bottom-sheet pickers (`SetupBottomSheetDialogFragment`, `SignInBottomSheetDialogFragment`) deleted at 100% rollout.
- **iOS:** existing SwiftUI + TCA `Welcome.swift`; only the bottom CTA region swaps (`signInOptions` replaces `ctaButtons`), Technicians moves to a top-bar item, plus an account-linking footnote below the buttons. Interactions: Technicians → `SSOCoordinator`; Amazon sign-in → `AmazonLoginCoordinator` (retail handle); Email or phone → `AmazonLoginCoordinator.privateAccountPool(loginSessionID:)`. Flag read as shared state so the toggle takes effect without relaunch. `WelcomeActionSheetViewController` removed at 100%.
- **Analytics:** each entry tracks `LoginAnalyticsEvents.StartedLogin` with the authentication type (`amazonSignIn` / `emailOrPhoneSignIn` / `technicianSignIn`).

## UnifiedCX vs Separate Sign-Up/Sign-In

Amazon Identity is moving to **UnifiedCX** — a single entry point where the user enters email/phone and the system determines new vs existing. The PAP uses UnifiedCX by default; separate paths require extra Identity work. **Pending Product alignment (target: June 15).** Affects duplicate-account-detection logic and mobile UX. SSO routing for B2B2X partners (e.g., T-Mobile) is out of scope but would be affected.

**Duplicate account detection:** when AuthPortal creates a new PAP account and the resolved email/phone belongs to an existing Amazon-login user, cloud deletes the new PAP account (`DeleteAccountV2`) and returns `error.form.email.unavailable` / `error.form.phone.unavailable` (same as today's `/2.2/register`).

## Migration Options (decision)

| Option | Summary | Trade-off |
|--------|---------|-----------|
| **Option 1: Hard cutover with throttle** | Backfill all credentials, ship AuthPortal-capable app, flip a kill switch | Clean single auth path; but min app version required and forced-update campaign; risk concentrated in one moment |
| **Option 2: Gradual sunset** | Both auth paths coexist; new versions use PAP, old use eero auth; deprecate over time | No forced lockout, lower risk; but two systems to maintain, credential-sync complexity, long tail |

Regardless of option, the migration is **never fully complete** because partners keep creating eero users.

## Non-Goals

- Amazon-login users, SSO users (unaffected)
- Password support at launch (passwordless/OTP-only preserved)
- Passkey support at launch (future, enabled by this foundation)
- Third-party login (Google, Apple) — future via FederationService

## Success Metrics

- 100% of active eero auth users have a PAP account with CID mapped.
- Post-cutover login/registration success rate ≥ baseline ([User Logins Grafana](https://grafana-prod.e2ro.com/d/xIup3vaWk/user-logins)).
- No increase in auth error rates or CS contacts ([API Stats by Route](https://grafana-prod.e2ro.com/d/mCmExpWmk/api-stats-by-route)).
- **Done when:** eero auth login/register endpoints are decommissioned and all auth flows go through AuthPortal. (P1: CS verification off Twilio, Twilio relationship ended.)

## References

- [Mobile Tech Spec (local copy)](./references/PAP-Migration-Mobile-Tech-Spec.md) — author: Adauton Heringer
- [Mobile Tech Spec (Google Doc)](https://docs.google.com/document/d/1yZ3M5ze90yTmJ3I15H0F2co84VP6EloHuWBLNR28zrg/edit)
- [ERD](https://docs.google.com/document/d/1mGoR0uoGjoKeqOD5w3BS-kvnyluwjPMsI-qC91rXLUQ/edit)
- [SSO Figma](https://www.figma.com/design/WjOfX2phnwdS2Go0iPDNSY/SSO?node-id=738-1863)

# Architecture

## High-Level Flow (PAP Login)

```
Client (MAP SDK / AuthPortal web view) → Amazon AuthPortal → MAP token
   → POST /2.3/login (eero user-service)
      → Aztec verifyAccessTokenForPermissions (MAP token → CID)
      → AddressService getContactInfo (CID → name/email/phone)
      → lookup users table by pap_customer_id
      → create eero session → return eero user token
```

1. Client opens AuthPortal (web view on web; MAP SDK web view on mobile).
2. User enters email/phone claim, receives OTP from Amazon Identity, verifies.
3. AuthPortal returns a **MAP token** to the client.
4. Client calls `POST /2.3/login` with the MAP token (+ optional `is_pro`).
5. user-service validates the token via **Aztec** → CID.
6. user-service fetches contact info via **AddressService** → name, email, phone.
7. Looks up `users` row by `pap_customer_id`.
   - **Found:** create verified session, return user token.
   - **Not found (new AuthPortal user):** create `users` row with CID + `pap_linked_at`, then session + token.

This mirrors the existing Amazon-login flow. A single route handles consumer and org logins; `is_pro=true` additionally verifies org membership, enforces the IP allowlist, and creates an Insight-enabled session.

## Terminology

| Term | Definition |
|------|------------|
| **PAP** | Private Account Pool — eero's namespace in Amazon Identity's data store |
| **CID** | Customer ID — globally unique account identifier in Identity |
| **MAP** | Mobile Authentication Platform — Amazon SDK; renders AuthPortal in a WebView |
| **AuthPortal** | Amazon's auth web UI (sign-in, registration, forgot password, account maintenance) |
| **Association Handle** | OpenID `assoc_handle` — client config identifying the account pool |
| **Aztec** | Identity backend that validates MAP tokens (token → CID) |
| **AddressService** | Identity read API (CID → name/email/phone) |
| **CES** | Customer Event Service — publishes credential-change events |
| **MOA / EOA / MCA** | Mobile-Only / Email-Only / Multi-Claim passwordless account |
| **DKV** | Dynamic Key-Value store used by eero cloud for runtime config |
| **UnifiedCX** | Single entry point UX: user enters email/phone, system decides new vs existing |

## Endpoints

### New PAP login route — `POST /2.3/login`

Replaces `/2.2/login`, `/2.2/register`, and `/2.2/pro/login`. Delivered by eero `user-service`.

| Element | Description |
|---------|-------------|
| Body | `auth_token` (required; URL-encode the `\|` character). `is_pro` optional (consumer omits or sends `false`). |
| Header | `X-Client-Device-Id: <IDFV on iOS, app-scoped UUID on Android>` |
| 200 | `{ "data": { "user_token": "…", "is_new_user": <bool> } }` — client stores `user_token`; `is_new_user` drives the marketing-consent fallback prompt. |

The `/2.2/login/amazon` retail call is **preserved** — retail Amazon-login and legacy-welcome (`feature_203=false`) clients continue to use it. Both endpoints coexist during rollout.

### Identity APIs (external)

| API | Purpose |
|-----|---------|
| **CreateAccountV2** | Create passwordless account, returns CID. Claim sets: `EmailAddressCreateAccountClaimSet`, `MobileNumberCreateAccountClaimSet` |
| **SetEmail / SetMobile** | Update email/phone for an existing CID |
| **DeleteAccountV2** | Soft-delete a PAP account (CID, Reason, Requestor) |
| **Aztec verifyAccessTokenForPermissions** | Validate MAP token → CID |
| **AddressService getContactInfo** | CID → name, email, phone |

## MAP Configuration (per-call, mobile)

Retail Amazon-login uses each platform's existing global MAP config. The PAP flow overrides parameters per-call, so retail is unaffected.

| Parameter | Retail (existing) | PAP (new) |
|-----------|-------------------|-----------|
| `associationHandle` | `amzn_eero_mobile_android_us` | `amzn_eero_mobile_us` |
| AuthPortal domain | MAP default (`www.amazon.com`) | `ap.account.eero.com` |
| PageId | default (= association handle) | pass `null` (uses default) |

- **Android:** per-call `AuthParameters` struct in `android-map-lib`; `associationHandle` → `openid.assoc_handle` on inner OpenID bundle; domain → `KEY_SIGN_IN_ENDPOINT` on outer options bundle (bare hostname). Do **not** set `KEY_REGISTRATION_DOMAIN` (Panda `getPandaHost()` rejects non-allowlisted values).
- **iOS:** per-call options dictionary (`MAPKeyOpenIdAssociationHandle`, `MAPKeyAuthPortalDomain`) via a new `EeroWebLoginCoordinator`; globals in `AmazonLogin.swift` stay untouched. AuthPortal domain must be a bare hostname.
- **AuthPortal proxy:** reverse proxy under `auth.eero.com` / `ap.account.eero.com` forwards `/ap/*` to AuthPortal's backend. Provides branded URL + cookie scoping. eero does not host AuthPortal's JS/assets.

## Feature Flags

Two independent, version-gated flags read from the unauthenticated `GET /2.2/app_configuration`.

| Flag | Renamed to | Min version | Behavior when `true` |
|------|-----------|-------------|----------------------|
| `feature_203` | `papLoginRequired` (was `AuthXUpgradeMessagingFeatureToggle`) | `26.14.0` (table) / full wiring ships 26.12 | Renders `WelcomeFragmentV2` / `WelcomeViewV2`; routes email/phone taps to the PAP flow |
| `feature_204` | `forceAppUpgradeToUsePap` (was `AuthXLoginFeatureToggle`) | moved `26.12.0` → `26.10.0` | 26.10/26.11 clients show `UpdateRequiredFragment` (Android) / `ErrorViewController` (iOS) before credential entry |

- `feature_203` gains `X-Client-Device-Id` bucketing (base class → `AppVersionAndDeviceIdDependentFeatureFlag`) for percentage rollout via `HASH_SHARD` throttle.
- JSON field names (`"feature_203"`, `"feature_204"`) are unchanged on the wire; renames are code-level in 26.12+.
- Server-side kill switches: `eeroAuthLoginDisabled` (→ `/2.2/login` + `/2.2/pro/login` return 404) and `eeroAuthRegistrationDisabled` (→ `/2.2/register` returns 404). Error: `error.login.upgrade_required`.

## Components

| Component | Role |
|-----------|------|
| **user-service** | Hosts `/2.3/login`; synchronous PAP account creation on registration routes |
| **authenticationlib** | Shared lib: all PAP sync logic (CreateAccountV2, SetEmail, SetMobile, DeleteAccountV2, CES processing) |
| **admin-service** | Identity writes via authenticationlib; hosts backfill + reconciliation routes |
| **monolithconsumers** | CES SQS consumer — decrypts credential-change events, updates `users` table |
| **AuthPortal proxy** | Reverse proxy under auth.eero.com for `/ap/signin`, `/ap/register` |
| **CES** | External — publishes `SetEmailEvent`, `SetMobileEvent`, `NameChangeEvent` to SNS → SQS |

No new microservice is created; all PAP sync logic lives in `authenticationlib`.

## Data Model Changes

Two new nullable columns on the `users` table:

- `pap_customer_id` — the PAP CID. Set during backfill or at first PAP login for new users.
- `pap_linked_at` — timestamp of backfill; gates whether eero → PAP sync fires on credential changes.

`amazon_customer_id` remains for Amazon-login users. A user never has both `pap_customer_id` and `amazon_customer_id` set.

## Credential Sync (Bidirectional, during transition)

- **eero → PAP (synchronous):** old-app credential changes call `SetEmail`/`SetMobile` on the PAP (only when `pap_linked_at` is set).
- **PAP → eero (event-based):** AuthPortal changes → CES event → monolithconsumers polls SQS, decrypts (Odin), filters by `accountPoolName`, calls AddressService for the new value, updates `users`. Mean latency ~1 min (p99 SLA <5 min).
- **Deletion (eero-initiated only):** soft-delete `users` row + `DeleteAccountV2`. AuthPortal has no self-service deletion.

## References

- [Mobile Tech Spec](https://docs.google.com/document/d/1yZ3M5ze90yTmJ3I15H0F2co84VP6EloHuWBLNR28zrg/edit)
- [Cloud HLD](https://docs.google.com/document/d/1tllKzECjAGPHwXNzsTnGXMsy8FusrG1WC98eSnV4pis/edit)
- [ERD](https://docs.google.com/document/d/1mGoR0uoGjoKeqOD5w3BS-kvnyluwjPMsI-qC91rXLUQ/edit)
- CR-297076250 — [MAPiOSLib] Allowlist eero PAP AuthPortal domains

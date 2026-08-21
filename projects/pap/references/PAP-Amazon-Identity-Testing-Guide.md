# PAP ↔ Amazon Identity Testing Guide

How to test eero's PAP integration with Amazon Identity — **AuthenticationService**    
(account + email + mobile claims), **AztecService** (token verification), and    
**AddressService** (contact info) — over CloudAuth, via the admin `/debug/pap/*` endpoints.

## Prerequisites

- A deployment with the PAP psenv configured for the pool under test (see below). Ephemeral dev works.
- `$EPH_HEADER` — the eph routing header. `$ADMIN_TOKEN` — an admin token with the TPI role.
- For `is_authorized`: an at-\* (web) token + client context (see [Getting an at-\* token](https://eeroinc.atlassian.net/wiki/spaces/CLOUD/pages/5599264957/PAP+Amazon+Identity+Testing+Guide#Getting-an-at-*-token)).

## Pool configuration (psenv)

Each pool maps to an Identity fabric. Set these before deploying/kicking the pod.

| Var | Devo pool (Identity **beta**) | Stage pool (Identity **prod**) | Prod pool (Identity **prod**) |
| --- | --- | --- | --- |
| `PAP_ACCOUNT_POOL_NAME` | `eero` | `eeroStage` | `eero` |
| `PAP_MARKETPLACE_ID` | `A2360Z5KKDC85H` | `A10ZONQX51YR1E` | `ATA3AUUGY6TS5` |
| `PAP_AUTHENTICATION_SERVICE_QUALIFIER` | `Base.Native.Beta` | \*(unset → \*`Base.Native.Prod`) |  |
| `PAP_ADDRESS_SERVICE_STAGE` | `native` | \*(unset → \*`native`) |  |
| `PAP_ADDRESS_SERVICE_REGION` | `test.USAmazon` | \*(unset → \*`prod.USAmazon`) |  |
| `PAP_AZTEC_REGION` | `test.USAmazon` | \*(unset → \*`prod.USAmazon`) |  |

> The reference.conf defaults target Identity **prod**. For the stage pool, set pool + marketplace and **unset** the devo overrides. Tokens/customerIds must match the fabric — devo uses beta tokens ([development.amazon.com](http://development.amazon.com)); stage uses prod tokens ([amazon.com](http://amazon.com)).

## Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/debug/pap/create_account` | Create a PAP account (email claim) |
| POST | `/debug/pap/set_email` | Set/replace the email claim |
| POST | `/debug/pap/set_mobile` | Set the mobile claim |
| POST | `/debug/pap/verify_token` | Verify a MAP access token (Aztec) |
| POST | `/debug/pap/is_authorized` | Verify an at-\* token (Aztec IsAuthorized) |
| GET | `/debug/pap/get_contact_info/:customer_id` | Read contact info (Address) |
| POST | `/2.3/login` | PAP login (**user** service, not a debug route) — see Logging in via `/2.3/login` |

> **Quoting gotcha:** the `eero api admin curl` wrapper coerces bare values, mangling `+` and appending `.0` to numeric-looking input. Wrap any value containing`+`, `/`, `=` in inner quotes — e.g. `-d 'phone_number="+14085550100"'`, `-d 'token="Atza|…"'`. (`%2B` does **not** work.)

## Test steps

Examples below are from the **devo** pool. Repeat identically for the stage pool after flipping psenv.

### 1. Create account

`name` is required. Add `create_user=true` to also insert an eero `users` row keyed by the new CID (used by the login flow below); the response then includes `userId`.

```bash
eero api admin curl -X POST -H "$EPH_HEADER" -H "cookie:session=$ADMIN_TOKEN" \
  -d 'email=burak-20260727.1906@eero.com' \
  -d 'name=burak-20260727.1906' \
  /debug/pap/create_account
```

```json
{ "meta": { "code": 200 }, "data": { "customerId": "AAWPL1CUMH168", "userId": null } }
```

409 `error.auth.external_duplicate_account` if the email already exists.

### 2. Set email

```bash
eero api admin curl -X POST -H "$EPH_HEADER" -H "cookie:session=$ADMIN_TOKEN" \
  -d 'customer_id=AAWPL1CUMH168' -d 'email=burak-20260727.1906-1@eero.com' \
  /debug/pap/set_email
```

```json
{ "meta": { "code": 200 }, "data": { "status": "ok" } }
```

### 3. Set mobile

Phone must be E.164 and **quoted** (see gotcha).

```bash
eero api admin curl -X POST -H "$EPH_HEADER" -H "cookie:session=$ADMIN_TOKEN" \
  -d 'customer_id=AAWPL1CUMH168' -d 'phone_number="+14085550100"' \
  /debug/pap/set_mobile
```

```json
{ "meta": { "code": 400, "message": "error.auth.external_invalid_claim" } }
```

> ⛔ **Currently blocked (Identity limitation).** The earlier `setMoible`→`setMobile` AAA grant typo is **resolved** — CloudAuth now authorizes the call. But `setMobile` still fails: non-CN pools reject setting a mobile claim unless the phone is already a **CIS-verified** contact (`InvalidAttributeException(UnverifiedMobile)`). `withEnforceVerification(false)` is **CN-only** and ignored for the eero pool, so there's no way to attach a pre-verified mobile via `setMobile` as-is. Unblocking needs an Identity-side mechanism (pool policy/weblab to relax phone verification like CN, or a verification-proof / verified-claim-at-creation path) — open question.

### 4. Get contact info

```bash
eero api admin curl -H "$EPH_HEADER" -H "cookie:session=$ADMIN_TOKEN" \
  /debug/pap/get_contact_info/AAWPL1CUMH168
```

```json
{ "meta": { "code": 200 }, "data": { "fullName": "" } }
```

> Connectivity works, but `fullName`/email/phone are **UCI (Restricted)** fields and come back empty until the calling app is **RED-certified + UCI allow-listed** with AddressService.

### 5. Verify at-\* token (IsAuthorized)

```bash
eero api admin curl -X POST -H "$EPH_HEADER" -H "cookie:session=$ADMIN_TOKEN" \
  -d 'token="Atza|…"' \
  -d 'client_context="131-1111111-1111111"' \
  /debug/pap/is_authorized
```

```json
{ "meta": { "code": 200 }, "data": { "customerId": "AAWPL1CUMH168" } }
```

Returns the customerId the token belongs to. Wrong pool/expired/invalid → `error.auth.external_token_*`.

### 6. Verify MAP token

```bash
eero api admin curl -X POST -H "$EPH_HEADER" -H "cookie:session=$ADMIN_TOKEN" \
  -d 'token="<MAP access token>"' \
  /debug/pap/verify_token
```

Same shape as `is_authorized`. Needs a **MAP access token** (not an at-\* token); untested pending an easy way to mint one.

## Getting an at-\* token

`is_authorized` needs an at-\* (website) token + its client context (ubid), from the **same** browser session:

1. Open an **incognito** window and go to the eero AuthPortal sign-in URL for the pool:
    1. Devo: `https://development.amazon.com/ap/signin?server=%2Fap%2Fsignin%3Fie%3DUTF8&openid.assoc_handle=amzn_eero_mobile_us&openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.return_to=https%3A%2F%2Fdevelopment.amazon.com%2Fap%2Ftests%3FreturnFromLogin%3D1&marketPlaceId=A2360Z5KKDC85H&pageId=pavaibha_eero_test&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&rmrMeStringID=ap_rememeber_me_default_message`
    2. Stage: `https://ap.stage.payments.e2ro.com/ap/signin?openid.assoc_handle=amzn_eero_mobile_dogfood_us&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&pageId=amzn_eero_mobile_dogfood_us`
2. Click **Forgot password**, enter the test OTP `112233` for Devo, or the real one emailed to you for the stage pool, set any new password.
3. Signed in → **DevTools (Cmd+Opt+I) → Application → Cookies → the amazon domain**:
    - `at-tacbus` → `token` (the at-\* token; suffix is marketplace-specific)
    - `ubid-tacbus` → `client_context` (the ubid; must share the same suffix as the `at-` cookie)
4. This is a web (`at-*`) token → use `/is_authorized` (not `/verify_token`, which needs a MAP token).

> `at-*` is HttpOnly, so read it from the Cookies panel, not `document.cookie`. Devo tokens only verify against Identity beta; use `amazon.com` (prod) tokens for the stage pool.

## Logging in via `POST /2.3/login`

`/2.3/login` is the real PAP login route on the **user** service (not a debug route). It validates the token via Aztec `IsAuthorized` (accepts both MAP and at-\* tokens), looks the user up by `pap_customer_id`, and returns an eero user token. It replaces `/2.2/login` (consumer) and `/2.2/pro/login` (pro) via an optional `is_pro` flag. Of the PAP psenv vars, only `PAP_AZTEC_REGION` matters for this route.

End-to-end flow (devo pool):

### 1. Create a PAP account + eero users row

Use `create_user=true` so create\_account also inserts a `users` row keyed by the new CID (`pap_customer_id`, verified email, `pap_linked_at=now()`). Existing-users-only lookup means the login won't find the user without this row. `name` is required.

```bash
eero api admin curl -X POST -H "$EPH_HEADER" -H "cookie:session=$ADMIN_TOKEN" \
  -d 'email=burak+20260729.1707@eero.com' \
  -d 'name=burak+20260729.1707' \
  -d 'create_user=true' \
  /debug/pap/create_account
```

```json
{ "meta": { "code": 200 }, "data": { "customerId": "A3FOL6RWCQ18O7", "userId": 157361 } }
```

### 2. Get an at-\* token for that email

Sign in at the devo AuthPortal with that email (OTP `112233`) and read `at-tacbus` (→ `auth_token`) + `ubid-tacbus` (→ `ubid`) from DevTools — see [Getting an at-\* token](https://eeroinc.atlassian.net/wiki/spaces/CLOUD/pages/5599264957/PAP+Amazon+Identity+Testing+Guide#Getting-an-at-*-token).

### 3. Call `/2.3/login`

Quote `auth_token` (contains `|`). `ubid` is optional; `is_pro` defaults to `false` (consumer).

```bash
eero api user curl -H "$EPH_HEADER" -X POST /2.3/login \
  -d 'auth_token="Atza|…"' -d 'ubid="131-4717301-6522147"'
```

```json
{ "meta": { "code": 200 }, "data": { "user_token": "4047784|…" } }
```

200 + `user_token`, plus a `set-cookie: s=…` header on the consumer path (`is_pro=false` → `UserToken`, with cookie). For the pro/Insight path add `-d 'is_pro=true'`: the response omits the cookie (`UserTokenNoCookie`), enforces the org IP allowlist, and returns **403** `error.auth.external_forbidden` for non-Insight users.

Error mapping:

| Condition | Result |
| --- | --- |
| Unknown CID (no eero row) | `error.login.unknown` |
| Token expired / invalid / wrong pool | 400 `error.auth.external_login_failed` |
| Aztec unavailable / permission denied | 503 `error.auth.external_service_unavailable` |
| `is_pro=true`, non-Insight user | 403 `error.auth.external_forbidden` |
| `is_pro=true`, org IP not allowlisted | 403 `error.auth.external_unauthorized_ip` |

## Known gaps as of Jul 29, 2026

Confirmed against **both** the devo and stage pools unless noted.

- `/2.3/login` — the **consumer path** (`is_pro=false` → 200 + `user_token` + session cookie) is validated end-to-end in **devo** (Aztec `IsAuthorized` → CID → `getByPapCustomerId` → session). The `is_pro=true` / 403 paths are not yet exercised on eph: the eph pods run commit `89663d4`, which predates the final `is_pro`/403 logic (`71859e3`) — redeploy on `71859e3` to test them.
- `set_mobile` — the earlier `setMoible`→`setMobile` AAA grant typo is **resolved** (CloudAuth now authorizes the call), but setting a mobile is still blocked by an **Identity limitation**: non-CN pools require a **CIS-verified** phone (`InvalidAttributeException(UnverifiedMobile)`), and `withEnforceVerification(false)` is CN-only. Backfilling a pre-verified mobile via `setMobile` isn't possible as-is — needs an Identity mechanism (pool policy/weblab, or a verification-proof path). Same bucket as the `get_contact_info` UCI gate and Sandfire deletion.
- `get_contact_info` UCI fields — connectivity works in both pools, but `fullName`/email/phone come back empty until the calling app is **RED-certified + UCI allow-listed** with AddressService.
- `verify_token` — not verified with a real token in either pool; needs a **MAP access token**. A dummy token correctly returns `external_token_invalid` (400), confirming the Aztec path is reachable + granted.
- **Stage** `is_authorized` / `verify_token` with a real token — not run yet: there's no working prod AuthPortal test sign-in URL to mint an at-\* token for an eeroStage account (requested from Identity). Real OTPs can be emailed/texted for sign-in (untested), so this should be unblocked once a valid prod URL is available. Dummy tokens return `external_token_invalid` (400) — proving Aztec **prod** is reachable + granted for both token operations — and the full happy path is proven in **devo** (`is_authorized` → 200 with the correct customerId).

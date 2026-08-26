# PAP Test-User Creation Runbook (stage, self-service)

**Verified Aug 26, 2026.** Lets QA create PAP test users in the `eeroStage` pool without depending on cloud per-account.

## Prerequisites

1. **TPI admin role** (or super user) on your stage admin identity. Without it, `create_account` returns `403 Insufficient permission`. Granted via the stage admin panel role grants (Burak/cloud). Current roles like `cx-sme`, `isp`, `qa-stage`, `commerce`, `cloud-on-call`, `group-manager` do **not** include it — TPI must be added explicitly.
2. **An admin token**, sent as the **`X-Admin-Token`** header (NOT `cookie: session=…`).
   - Generate one: log into `https://admin.stage.e2ro.com/` → DevTools → Application → Cookies → copy the `token` value (tick **"Show URL decoded"**). The old `POST /login` token endpoint is deprecated.
   - Henrique's `.zshrc` `ADMIN_TOKEN` works as `X-Admin-Token` once TPI is granted.

## Create a user

```bash
curl -X POST \
  -H "X-Admin-Token: $ADMIN_TOKEN" \
  --data-urlencode "email=henrique+paptest-$(date +%Y%m%d.%H%M%S)@eero.com" \
  --data-urlencode "name=henrique-paptest" \
  --data-urlencode "create_user=true" \
  "https://api-admin.stage.e2ro.com/debug/pap/create_account"
```

- `create_user=true` also inserts the eero `users` row (`pap_customer_id`, verified email, `pap_linked_at=now()`) so `/2.3/login` can find the user.
- Response: `{"meta":{"code":200},"data":{"customerId":"A3ERLW9CAV3XQ7","userId":177214}}`. **Persist `customerId` + email** — account details aren't recoverable later.
- Use an email that routes to a mailbox you control (`henrique+tag@eero.com`). `eero.com` is Google Workspace with `+` sub-addressing, so the sign-in OTP lands in your `henrique@eero.com` inbox.

## Signing in with a created account

- The account is **passwordless** (email claim) → AuthPortal sign-in uses an **OTP** (emailed to the account address → your Gmail inbox).
- For repeatable **password** login: set a password once via AuthPortal "forgot password" (one OTP), then sign in with email+password thereafter (this is how the shared `123123` account was set up).
- Stage AuthPortal: `https://ap.stage.payments.e2ro.com/ap/signin` (assoc_handle `amzn_eero_mobile_dogfood_us`, marketplace `A10ZONQX51YR1E`).
- Then `POST https://api-user.stage.e2ro.com/2.3/login` with the `at-*`/MAP token (`auth_token`) → `user_token`.

## Auth gotchas (learned the hard way)

- Use **`X-Admin-Token: <token>`** — NOT `cookie: session=<token>` (that returns 401).
- Status meaning: **401** = token not recognized as admin; **403** = valid admin but missing TPI role; **200** = success.
- Form values containing `+ / = @` are URL-encoded automatically by `curl --data-urlencode` (fine). The `eero api` CLI wrapper's `%2B`-doesn't-work gotcha does **not** apply to raw curl.

## What this does NOT solve

- These are **normal-risk** accounts → automated/headless or fresh-device sign-in can hit **CVF**, and AWS Device Farm (a new device each run) trips CVF + new-device gates (Not-Me/TIV). For CVF-free **programmatic** sign-in at scale you need **low-risk accounts (TipocaService)** in the PAP — pending validation that Tipoca supports marketplace `A10ZONQX51YR1E`. See [kamino-api-shadow-coordination.md](./kamino-api-shadow-coordination.md).
- There is **no programmatic OTP-retrieval API** for PAP in stage/prod (by design). Devo uses a hardcoded OTP `112233`.

## Endpoints / hosts

| Purpose | Host / route |
|---------|--------------|
| Admin debug routes | `https://api-admin.stage.e2ro.com/debug/pap/*` — `create_account`, `set_email`, `set_mobile` (blocked), `verify_token`, `is_authorized`, `get_contact_info` |
| PAP login | `https://api-user.stage.e2ro.com/2.3/login` |
| Admin panel (mint token) | `https://admin.stage.e2ro.com/` |

## Created so far

| Email | CID | userId | Created |
|-------|-----|--------|---------|
| `henrique+paptest-20260826.161046@eero.com` | `A3ERLW9CAV3XQ7` | 177214 | Aug 26, 2026 |

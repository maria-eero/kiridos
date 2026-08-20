# Processes

## Migration Phases

### Phase 1: Backfill
Migrate all existing eero auth credentials to the PAP **before** enabling PAP auth for any client.

- Add `pap_customer_id` + `pap_linked_at` columns.
- Admin-service route accepts a range of user IDs, migrates any not already migrated (checks `pap_linked_at`): reads credentials, calls `CreateAccountV2`, stores CID. MCA = `CreateAccountV2` with email claim + `SetMobile` for phone.
- Executed via script (local/dev desktop) following MCM templates for stage then prod. Rate limits for `CreateAccountV2` TBD empirically in preprod.
- When backfill starts, a throttle gates synchronous PAP creation on registration routes: new users via `/2.2/register` and `/2.2/organizations/self/register` (Frontier) also get PAP accounts synchronously. If `CreateAccountV2` fails, the whole registration fails (no orphans).
- Reconciliation pass after backfill compares eero vs PAP credentials via AddressService, fixes mismatches.
- **Exit:** all eero auth users have a PAP CID mapped; registration throttle on; bidirectional sync ready.

### Phase 2: Transition
Both auth paths live. New client versions support both and check a server-side throttle. Bidirectional credential sync active.

- Integrate MAP SDK (iOS + Android); include `X-Device-Id` header in all requests from PAP-capable versions.
- Update account.eero.com + Insight to AuthPortal web view.
- Implement rollout flags (FF1/FF2) + percentage device throttle; duplicate-account detection; Amazon-login migration (drop CID + `DeleteAccountV2`).
- **Dashboards + alarms before first client goes live** covering every user-facing failure mode (see below).
- **PoC: validate Kamino + APEX device registration produces MAP tokens for the eero PAP marketplace — blocker for stage automation.**

### Phase 3: Final Cutover
Go/no-go criteria:
- PAP login success rate + latency at parity with eero-auth baseline.
- CES sync lag within SLA (mean <1 min, p99 <5 min).
- Zero critical PAP-auth incidents in a defined incident-free window.
- Sufficient app-version adoption on PAP-capable versions.

Then: in-app "update" messaging → review adoption → flip cutover flag → old versions can no longer use eero auth (see `UpdateRequired` / generic 404 on very old versions) → gate `eeroAuthLoginDisabled` / `eeroAuthRegistrationDisabled` (404 `error.login.upgrade_required`).

### Phase 4: Cleanup
- Decommission `/2.2/login`, `/2.2/login/verify`, `/2.2/register*`, `/2.2/pro/login`.
- Remove eero → PAP sync (no longer needed).
- CES subscription stays active (PAP is sole source of truth; `users` table kept in sync for transactional email, CS tooling, internal services).

## Rollout Strategy

Version-gated feature flags (FF1/FF2 via `app_configuration`) + percentage-based device-ID hash throttle (`HASH_SHARD`) for mobile, server-side switching for web.

- **FF1 (deprecation warning):** shows "this version will stop working"; old flow still works.
- **FF2 (AuthX switch):** switches client to AuthPortal/MAP.
- Client identified by **User-Agent** for version gating (UX only — no resource authZ; MAP token validated server-side by Aztec regardless).
- **Rollout order:** account.eero.com → Insight (server-side, no version dep) → iOS/Android (version-gated + device throttle, ramp 5% → 100%).
- Device ID: IDFV (iOS, resets on reinstall) / app-scoped ID (Android, resets on factory reset).

## Rollback Strategy

Every step is a two-way door via DKVs:
- FF1/FF2 → `enabledForPublic=false` or raise `minVersion` → clients revert on next `app_configuration` fetch.
- Device throttle → 0% → all mobile back to eero auth.
- Kill switch → false → old endpoints accept requests again.
- eero → PAP sync runs throughout, so `users` remains source of truth (no data loss on revert).
- Only irreversible step is Phase 4 (code removal), after extended stability.

## Release / Version Strategy

- Full `feature_203` wiring, `WelcomeFragmentV2`/`WelcomeViewV2`, PAP flow, and `/2.3/login` ship in **26.12**; both flags default off server-side.
- `feature_203` min version listed as **26.14.0** (flag table); `feature_204` min version moved **26.12.0 → 26.10.0**.
- Identity passwordless (EOA/MOA) dependency target: **Nov 2026**.

## Monitored Failure Modes (dashboards/alarms before go-live)

Token validation failures (expired/invalid/revoked/wrong pool); AddressService lookup failures; user-not-found after auth (backfill gap); duplicate-account detection; session creation failures; credential sync lag; registration failures with throttle on; `SetMobile` failures during MCA creation (partial state); kill switch enabled prematurely; MAP token format/transport errors; mobile SDK failures cloud can't see.

## Key Decisions

| Decision | Notes |
|----------|-------|
| Hybrid rollout (version-gated FF + device-ID hash throttle) | No server-side device-ID storage |
| Single `/2.3/login` route with `is_pro` | Replaces `/2.2/login`, `/2.2/register`, `/2.2/pro/login` |
| New `pap_customer_id` column (not reuse `amazon_customer_id`) | Keeps linking flow clean |
| **MAP for mobile, AuthPortal web view for web** | MAP uses web views internally (Goodreads precedent) |
| **Test accounts: Kamino over Tipoca** | Kamino + APEX yields MAP tokens for `/2.3/login`; Tipoca only vends auth tokens / at-main cookies (MAP out of scope). Pending validation vs eero PAP marketplace. See [test-user-login-concerns.md](./test-user-login-concerns.md) |
| Consent | Identity supports one required checkbox (ToS); marketing needs extra optional checkbox or post-reg opt-in |
| UnifiedCX vs separate sign-up/sign-in | Pending Product (target June 15) |

## Precedent: Goodreads PAP Migration

- Shadow-mode testing: backfill all credentials, run AuthService in parallel to verify correctness before cutover.
- Phased rollout by surface: web → mobile (MAP) → 3P login → cleanup.
- Real-time credential sync + CES during transition.
- Account deletion must cover both systems (GDPR/CCPA) — delete from `users` and PAP (`DeleteAccountV2`).
- CS tooling impact: Goodreads lost manual reset-link generation; eero's CS OTP flow needs preserving/replacing post-migration.

## References

- [Cloud HLD](https://docs.google.com/document/d/1tllKzECjAGPHwXNzsTnGXMsy8FusrG1WC98eSnV4pis/edit)
- [ERD](https://docs.google.com/document/d/1mGoR0uoGjoKeqOD5w3BS-kvnyluwjPMsI-qC91rXLUQ/edit)

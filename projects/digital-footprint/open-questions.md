# Digital Footprint — Open Questions

## Resolved

| # | Question | Status | Answer |
|---|----------|--------|--------|
| 11 | Can we retrieve OTP codes via admin API for test automation? | ✅ Resolved | Account email uses standard verification system — admin API (`GET /test/users/verify`) works. Custom/non-account email uses a DIFFERENT system — admin API won't work for those. |
| 12 | Once a custom email is verified, is it stored permanently? | ✅ Resolved | Account email verification persists. Custom/non-account email verification does NOT persist — it's one-time only. |
| 13 | Can we get test users configured for each scan result type? | ✅ Resolved | Need mock endpoints — no per-user deterministic results from real API. |
| 14 | Can we get test users with DFP permission disabled? | ✅ Resolved | Feature flag configurable per user ID. Jason will set up during automation process. |

## For Next Sync

| # | Question | Status | Context |
|---|----------|--------|---------|
| 1 | Is the info modal (ℹ️ icon) still in scope for 26.7? | TBD | Jason mentioned an info icon on the scan results screen that opens a modal explaining "what is a data exposure", "how can I get protected", basic FAQ. Copy was to be written by Jason and approved by legal. Duncan confirmed the pattern exists elsewhere in the app (e.g., Wi-Fi frequencies). Not visible in current Figma designs or Mobile Tech Spec. |
| 2 | Has legal reviewed and approved the final copy? | TBC | Multiple sync discussions mentioned needing legal sign-off on: summary strings, info modal content, and the "link out to Malwarebytes" copy. Jason was going to share the tech spec with security and legal. String freeze is June 3rd — all copy must be finalized by then. |
| 3 | Have execs (Nick Weaver / Mark Seabach) seen the current designs? | TBC | Amanda raised concern about avoiding the "health tab situation" where execs rejected months of work post-implementation because they never approved it. Nick Butler had previously defended the feature when Mark Seabach questioned its priority ("this has nothing to do with AI"). Amanda suggested spinning up a Slack thread with Nick Butler and Duncan to confirm exec visibility. |
| 4 | Is the summary string localization approach finalized? | TBD | Team decided mobile handles localization using a template like "Your personal information has been exposed in {X} breaches." This avoids cloud generating dynamic strings that are hard to localize. Need to confirm: is this the final approach? Are the string keys ready for string freeze (June 3rd)? |
| 5 | What is the "Scan now" vs "Next" button logic? | TBC | From Figma: bottom sheet shows "Scan now" (blue) when account email is prefilled, but shows "Next" (gray→blue) when user clears and types a custom email. Need to confirm: is this driven by whether the email matches the account email, or by whether the email field was manually edited? |
| 6 | What exactly determines High vs Medium risk level? | TBD | From transcripts: "High risk" was defined as Password, Location, and SSN exposures. "Medium risk" is online information only (online accounts, email). Need cloud team to confirm the exact mapping of categories → risk levels, as this drives the banner color (red vs yellow/orange). |
| 7 | Max breaches limit — is 100 confirmed? | TBC | Cloud Tech Spec states "Cloud will limit max number of breaches returned to 100." Need to confirm this is still the case and understand behavior when a user has >100 breaches (are they truncated? sorted by date? any indicator to user?). |
| 8 | Feature flag `feature_132` — is it already active in dogfood/staging? | TBC | From transcript: Jason mentioned the flag existed from last summer's attempt. Dalmo confirmed he's already using it on Android. Need to confirm it's enabled in test environments for QA. |
| 9 | Test accounts — are mock endpoints live? | TBD | Jason said the PR with sample/test endpoints would go in "in the next day or so" (from the sync). These return static JSON for mobile to test against while business logic is being built. Need to confirm: are they merged and available? What accounts should QA use? |
| 10 | Priority conflict — Virtual Agent vs Digital Footprint | TBC | Amanda told Jason to prioritize Virtual Agent over DFP if there's a conflict ("Virtual Agent is much harder, if we have to delay this project because of Virtual Agent, Virtual Agent has high priority"). Need to confirm: is this still the case? Any risk to DFP timeline? |
| 15 | Mock endpoints for deterministic scan results | TBD | Jason confirmed we need mock endpoints for automation. Need to know: when will they be available? What format? Can we get fixed responses per email address (e.g., test+highrisk@e2ro.com always returns high risk)? |
| 16 | Phone-only test account | TBD | Jason flagged that some accounts have phone only (no email). Need a test account with this setup for manual EDGE-05 testing. No existing automation test users have this configuration. |

## Context Sources

- **Transcript 1** (sync ~May 19): Lauren on PTO, Jason starting mock endpoints, Henrique planning test automation for P0/P1, Lauren updated Mobile Tech Spec (contextual warnings struck through), Duncan still getting design feedback
- **Transcript 2** (sync ~May 26): Jason's PR with sample answers ready this week, MWB mapping done, feature flag confirmed, design ready per Duncan, Android implementation started (Dalmo working on home card)
- **Transcript 3** (earlier design discussion): Decided on 8 categories (not showing PII), breach list with detail view, "have I been pwned" as reference, summary string approach, upsell always shown
- **Transcript 4** (earlier design/API discussion): API response structure decided (source_breachs array with full data per breach, no ID system needed), 6 exposed categories grid, zero state needed, loading state = spinner on button (P99 ~2s), localization approach for summary string
- **Transcript 5** (earlier sync): Permissions issue (feature flexibility doesn't support owner vs admin — Jason adding it), exec approval concern raised, feature flag from last summer, Virtual Agent priority conflict
- **Jason Slack (May 27)**: OTP for account email uses standard system (admin API works), custom email uses different system. Verified emails persist for account email only. Need mock endpoints for scan results. Feature flag configurable per user ID. Phone-only accounts exist but are rare.

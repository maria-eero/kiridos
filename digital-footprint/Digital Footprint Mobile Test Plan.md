# QA test plan

**Digital Footprint Mobile Test Plan**

**Version**: 1.5

**Authors**: Henrique  
**Template Version**: 3.1

*Note: Please read the test plan guidelines before starting to create your test plan*

## 1\. Introduction

Digital Footprint Native Scan integrates Malwarebytes' dark web scanning capabilities natively into the eero mobile app. The feature enables users to scan their email address for data breaches and compromised information, view exposed categories, browse breach details, and receive actionable recommendations. It gates access via feature flag (feature\_132), network capability, and owner permissions. Non-subscribers are presented with a contextual Plus upsell flow.

### 1.1 Links to Relevant Documentation

* PRD: [Digital Footprint PRD](https://docs.google.com/document/d/1UZuaRqTBeqP2i5JQn8BtpNnmhkDCKWdwxEDoeS_liN4/edit)
* Tech Spec: [Digital Footprint Mobile Tech Spec](https://docs.google.com/document/d/1yZ3M5ze90yTmJ3I15H0F2co84VP6EloHuWBLNR28zrg/edit)
* Cloud Tech Spec: [Digital Footprint Cloud Tech Spec](https://docs.google.com/document/d/1x3twjnG2ugyN1wBOStBr-gFXYb2PhTI57ZOTIj8CMi8/edit)
* JIRA Initiative: [SWS-31406: Digital Footprint Native Scan](https://eeroinc.atlassian.net/browse/SWS-31406)
* Figma: [Digital Footprint Scan Designs](https://www.figma.com/design/fQfUJ3k8r0XJxt0kQDkFzY/Digital-Footprint-Scan?node-id=155-15072&p=f&t=zgycNyayJJ8rsMgW-0)
* Confluence: [Digital Footprint Native Scan](https://eeroinc.atlassian.net/wiki/spaces/sws/pages/4875747344/Digital+Footprint+Native+Scan)
* TestRail: [Digital Footprint Test Cases (Suite 156, Section 593094)](https://testrail.eero.amazon.dev/index.php?/suites/view/156&group_by=cases:section_id&group_order=asc&display_deleted_cases=0&group_id=593094)

### 1.2 Feature Flag

| Feature Flag | User Roles | Description |
| :---- | :---- | :---- |
| feature\_132 | Network owner (residential) | Gates Digital Footprint scan feature. Requires `features.digital_footprint_scan.capability.capable == true` AND `permissions.digital_footprint_scan.create == true` |

### 1.3 eeroOS/App version

* **eeroOS**: All versions
* **iOS App**: 26.7
* **Android App**: 26.7

### 1.4 Milestone Breakdown

| Milestone | Scope | Story Points |
| :---- | :---- | :---- |
| iOS Implementation | Full DFP flow (home card, scan, results, breaches, upsell, analytics) | TBD |
| Android Implementation | Full DFP flow (home card, scan, results, breaches, upsell, analytics) | TBD |
| Cloud Implementation | API endpoints (scan, verify\_email, entitlements, permissions) | TBD |

### 1.5 In Scope

- Gate logic validation for feature flag, capability, and permission combinations
- Home card entry point (dark blue card with dismiss X, display logic, 30-day reappearance)
- Scan initiation bottom sheet (email prefill, "Scan now" CTA, "Next" button states)
- Email verification full screen (6-digit code input, "Resend code" link, "Verify and scan" CTA)
- Account email unverified flow (standard OTP verification before scan)
- Scan results screen (timeline view, risk summary banner, exposed categories, "View all" link)
- Risk level variants: High (red), Medium (yellow/orange), No risk (green), Protected (blue info)
- Plus upsell card ("Get covered with guided support" / "Protect your identity & devices")
- Free protection section (help articles with external link icons)
- Plus subscriber recommendations ("Protect your passwords" -> 1Password, "Protect your location" -> VPN)
- Breaches list screen (filtered by category and full list via "View all")
- Breach detail screen (with contextual upsell CTAs)
- Troubleshooting entry point ("Digital footprint scan" under "Tools & resources")
- Navigation bar ("Back" left, "Digital Fooprint" center, "scan"/"New scan" right action)
- Phone-only account behavior (no email associated)
- Analytics events for all entry points and actions
- Platform parity: iOS and Android

### 1.6 Out of Scope

- Contextual warnings through the app (Settings, Account Settings, Devices — deferred from Phase 1)
- Web payments portal (account.eero.com)
- eero.com and a.com marketing material updates
- eero Business (EB) users
- Plus upsell modal UI validation (already covered in existing test suite: `test_c967958`)

### 1.7 Sign off Criteria

- All P0 and P1 test scenarios pass on both iOS and Android
- No open Sev1 or Sev2 defects
- Feature flag correctly gates the feature
- Analytics events fire with correct properties on both platforms
- Platform parity confirmed (iOS/Android behavior matches)
- Risk summary banner displays correct level (High/Medium/No risk/Protected)
- All CTA button states (enabled/disabled) match Figma
- Regression suite passes with no new failures
- Accessibility audit passed (VoiceOver + TalkBack)

---

## 2\. Test Environment

### 2.1 Hardware Models and eeroOS version

| Hardware | eeroOS Version |
| :---- | :---- |
| Any eero gateway | Any |

### 2.2 Mobile Devices and OS version

| Platform | Device | OS Version |
| :---- | :---- | :---- |
| iOS | iPhone 14, iPhone 15 Pro, iPhone SE 3rd gen | iOS 16, iOS 17, iOS 18 |
| Android | Pixel 7, Samsung Galaxy S23, Samsung Galaxy A54 | Android 13, Android 14, Android 15 |

### 2.3 Network Configuration

- Standard residential network (single gateway) with owner account
- Network with Plus subscription active
- Network without Plus subscription

---

## 3\. Test Data

### 3.1 User type

| User Type | Role | Feature Flag | Capability | Permission | Expected Visibility |
| :---- | :---- | :---- | :---- | :---- | :---- |
| Standard owner (Plus) | Owner | feature\_132 ON | true | create=true | Full DFP flow |
| Standard owner (no Plus) | Owner | feature\_132 ON | true | create=true | Full DFP flow + upsell |
| Non-owner user | Invited | feature\_132 ON | true | create=false | No DFP UI |
| Admin user | Admin | feature\_132 ON | true | create=true | No DFP UI |
| EB (Business) user | Owner | feature\_132 ON | true | create=true | No DFP UI |
| Flag disabled user | Owner | feature\_132 OFF | true | create=true | No DFP UI |
| Not capable user | Owner | feature\_132 ON | false | create=true | No DFP UI |
| Phone-only user | Owner | feature\_132 ON | true | create=true | TBD (no email to scan) |

### 3.2 Scan result type

| Result Type | Risk Level | Icon Color | Banner Text | Cloud API Response |
| :---- | :---- | :---- | :---- | :---- |
| High risk | High | Red (!) | "Risk summary: High" | `scan_result` with high-risk categories (password, SSN, phone) |
| Medium risk | Medium | Yellow/Orange | "Risk summary: Medium" | `scan_result` with online information only |
| No risk | No risk | Green | "Risk summary: No risk" | `scan_result` with all zeros |
| Protected (Plus subscriber) | Protected | Blue (i) | "Risk summary: Protected" | `scan_result` with exposures + `identity_protection_enabled: true` |
| Non-subscribed high risk | High | Red (!) | "Risk summary: High" | `scan_result` with exposures + `identity_protection_enabled: false` |
| Unverified email | N/A | N/A | N/A | `email_unverified: true` |

### 3.3 Subscription Type

#### 3.3.1 Plus subscriber

- Active eero Plus with identity protection enabled (`identity_protection_enabled: true`)
- Active eero Plus with identity protection disabled (`identity_protection_enabled: false`)

#### 3.3.2 Non-subscriber

- No active subscription (upsell flow should be presented)

---

## 4\. Test Strategy

### 4.1 Manual Scope

- **Exploratory testing** of all screens and flows on both platforms
- **Gate logic validation** across all user type and feature flag combinations
- **End-to-end flow testing** for each user/subscription/risk level combination
- **Visual validation** against Figma designs (icon colors, banner styles, button states)
- **Edge cases**: network errors, validation boundaries, app lifecycle events, phone-only accounts
- **Analytics validation** using Charles Proxy / analytics debug tools
- **UI/UX consistency**: navigation, bottom sheet gestures, CTA state transitions
- **Accessibility** testing (VoiceOver/TalkBack, dynamic type)

### 4.2 Automated scope

#### 4.2.1 SDE

- Unit tests for gate logic (feature flag + capability + permission evaluation)
- Unit tests for email validation logic
- Unit tests for card dismissal/30-day reappearance logic
- Unit tests for risk level determination (High/Medium/No risk/Protected)

#### 4.2.2 QAE

- E2E automation for critical path flows, when and if feasible
- Each automated test follows: Login as user -> Navigate to DFP -> Verify flow
- Avoid duplicating existing Plus upsell modal tests (`test_c967958`)
- Regression automation for home screen and troubleshooting screen
- Feature flag gating tests (configurable per user ID via Jason)
- Scan result verification tests depend on mock endpoints availability

### 4.3 Regression

- Existing Plus subscription flows (`test_premium_features_screen.py`)
- Home screen card rendering (`test_home_screen.py` — other cards unaffected)
- Settings/Troubleshooting section (`test_help_tab_migration_troubleshooting_screen.py`)
- Identity Protection existing flows

---

## 5\. Test Tools

| Tool | Purpose |
| :---- | :---- |
| TestRail | Test case management — [Digital Footprint section](https://testrail.eero.amazon.dev/index.php?/suites/view/156&group_by=cases:section_id&group_order=asc&display_deleted_cases=0&group_id=593094) |
| Jira | Defect management and tracking |
| Charles Proxy | Analytics event validation, API request/response inspection |
| Xcode / Android Studio | Debug builds, feature flag toggling |
| Appium | E2E mobile automation framework (when feasible) |
| Accessibility Inspector / TalkBack | Accessibility validation |
| Admin API | OTP code retrieval for automation (`GET /test/users/verify`) |

## 6\. Test Scenarios for eero App

Each test scenario represents one complete user flow (login -> navigate -> verify). Scenarios are scoped to avoid overlap with existing automated coverage. All cases are tracked in [TestRail Suite 156, Section 593094](https://testrail.eero.amazon.dev/index.php?/suites/view/156&group_by=cases:section_id&group_order=asc&display_deleted_cases=0&group_id=593094).

| \# | TestRail | BRD/PRD | Requirements | Test Scenarios | Automation |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Feature Gating** |  |  |  |  |  |
| 1 | [C2946053](https://testrail.eero.amazon.dev/index.php?/cases/view/2946053) | GATE-01 | DFP not visible when feature flag is OFF | Login as owner with feature\_132 OFF -> Navigate to Home -> Verify no DFP card visible -> Navigate to Settings > Troubleshooting -> Verify no "Digital footprint scan" row | Candidate |
| 2 | [C2946054](https://testrail.eero.amazon.dev/index.php?/cases/view/2946054) | GATE-02 | DFP not visible when capability is false | Login as owner with feature\_132 ON but capability false -> Navigate to Home -> Verify no DFP card -> Navigate to Troubleshooting -> Verify no DFP row | Candidate |
| 3 | [C2946055](https://testrail.eero.amazon.dev/index.php?/cases/view/2946055) | GATE-03 | DFP not visible for non-owner | Login as non-owner (invited user) -> Navigate to Home -> Verify no DFP card -> Navigate to Troubleshooting -> Verify no DFP row | Candidate |
| 4 | [C2946056](https://testrail.eero.amazon.dev/index.php?/cases/view/2946056) | GATE-04 | DFP not visible for Admin user | Login as Admin user -> Navigate to Home -> Verify no DFP card -> Navigate to Troubleshooting -> Verify no DFP row | Candidate |
| 5 | [C2946057](https://testrail.eero.amazon.dev/index.php?/cases/view/2946057) | GATE-05 | DFP not visible for EB user | Login as eero Business user -> Navigate to Home -> Verify no DFP card -> Navigate to Troubleshooting -> Verify no DFP row | Candidate |
| 6 | [C2946058](https://testrail.eero.amazon.dev/index.php?/cases/view/2946058) | GATE-06 | DFP visible when all conditions met | Login as owner with feature\_132 ON, capable, permission granted -> Navigate to Home -> Verify DFP card visible with correct copy ("Your digital security may be compromised") -> Navigate to Troubleshooting -> Verify "Digital footprint scan" row under "Tools & resources" | Candidate |
| **Home Card Entry Point** |  |  |  |  |  |
| 7 | [C2946059](https://testrail.eero.amazon.dev/index.php?/cases/view/2946059) | HOME-01 | Home card opens DFP bottom sheet | Login as owner -> Verify DFP card on Home -> Tap card -> Verify "Digital Fooprint Scan" bottom sheet opens with email prefilled and "Scan now" CTA | Candidate |
| 8 | [C2946060](https://testrail.eero.amazon.dev/index.php?/cases/view/2946060) | HOME-02 | Home card dismiss via X persists | Login as owner -> Verify DFP card on Home -> Tap X button on card -> Verify card disappears -> Force close app -> Reopen -> Verify card still not visible | Not Automated |
| 9 | [C2946061](https://testrail.eero.amazon.dev/index.php?/cases/view/2946061) | HOME-03 | Home card reappears after 30 days | Login as owner -> Dismiss card -> Advance device clock 30 days -> Reopen app -> Verify card reappears | Not Automated |
| 10 | [C2946062](https://testrail.eero.amazon.dev/index.php?/cases/view/2946062) | HOME-04 | Home card hidden after completing scan | Login as owner -> Tap DFP card -> Complete scan flow -> Navigate back to Home -> Verify card no longer appears | Not Automated |
| **Scan Initiation — Account Email** |  |  |  |  |  |
| 11 | [C2946063](https://testrail.eero.amazon.dev/index.php?/cases/view/2946063) | SCAN-01 | Full scan with account email (happy path) | Login as owner -> Tap DFP card -> Verify bottom sheet with prefilled email and "Scan now" blue CTA -> Tap "Scan now" -> Verify scanning state ("Scanning...", placeholder cards) -> Verify results load | Candidate |
| 12 | [C2946064](https://testrail.eero.amazon.dev/index.php?/cases/view/2946064) | SCAN-02 | Bottom sheet email field interactions | Login as owner -> Tap DFP card -> Verify email prefilled -> Tap X to clear email -> Verify "Next" button is gray/disabled -> Type valid email -> Verify "Next" becomes blue/enabled -> Swipe down on grabber -> Verify sheet closes | Not Automated |
| 13 | [C2946065](https://testrail.eero.amazon.dev/index.php?/cases/view/2946065) | SCAN-03 | Invalid email validation | Login as owner -> Tap DFP card -> Clear email -> Type invalid email (e.g., "notanemail") -> Tap "Next" -> Verify inline validation error shown | Not Automated |
| 14 | [C2946259](https://testrail.eero.amazon.dev/index.php?/cases/view/2946259) | SCAN-04 | Account email unverified — verify then scan | Login as owner with unverified account email -> Tap DFP card -> Tap "Scan now" -> Verify OTP screen appears (standard verification) -> Retrieve code via admin API -> Enter code -> Verify scan proceeds to results | Candidate |
| **Scan Initiation — Custom Email with Verification** |  |  |  |  |  |
| 15 | [C2946066](https://testrail.eero.amazon.dev/index.php?/cases/view/2946066) | VER-01 | Email verification happy path | Login as owner -> Tap DFP card -> Clear email -> Enter different email -> Tap "Next" -> Verify "Verify email" screen with correct subtitle showing email -> Verify "Verify and scan" button disabled -> Enter 6 digits -> Verify button becomes blue -> Tap "Verify and scan" -> Verify scan proceeds to results | Not Automated |
| 16 | [C2946067](https://testrail.eero.amazon.dev/index.php?/cases/view/2946067) | VER-02 | Incorrect verification code | Login as owner -> Enter custom email -> Tap "Next" -> On verify screen enter wrong code -> Tap "Verify and scan" -> Verify error message shown -> Verify retry allowed | Not Automated |
| 17 | [C2946068](https://testrail.eero.amazon.dev/index.php?/cases/view/2946068) | VER-03 | Resend code and clear input | Login as owner -> Enter custom email -> Tap "Next" -> On verify screen tap "Resend code" -> Verify code resent -> Enter digits -> Tap X to clear -> Verify field cleared and button disabled again | Not Automated |
| **Scan Results — High Risk (Non-subscriber)** |  |  |  |  |  |
| 18 | [C2946069](https://testrail.eero.amazon.dev/index.php?/cases/view/2946069) | HIGH-01 | High risk results UI (non-subscriber) | Login as non-subscriber owner -> Complete scan -> Verify timeline: green "Scanned" -> yellow "X online information" -> red "X high-risk exposures" with "View all" -> Verify red banner "Risk summary: High" with (!) icon -> Verify upsell section -> Verify "Free protection" section with help articles | Candidate |
| 19 | [C2946070](https://testrail.eero.amazon.dev/index.php?/cases/view/2946070) | HIGH-02 | Tap "Get protected now" opens upsell | Login as non-subscriber owner -> Complete scan (high risk) -> Tap "Get protected now" -> Verify Plus IAP upsell modal opens -> Close upsell -> Verify returns to results | Not Automated |
| 20 | [C2946071](https://testrail.eero.amazon.dev/index.php?/cases/view/2946071) | HIGH-03 | Tap "View all" opens breaches list | Login as non-subscriber owner -> Complete scan (high risk) -> Tap "View all" -> Verify breaches list screen loads with all breaches | Not Automated |
| 21 | [C2946072](https://testrail.eero.amazon.dev/index.php?/cases/view/2946072) | HIGH-04 | "New scan" nav and Plus recommendations | Login as non-subscriber (returning user) -> Navigate to results -> Verify nav bar shows "New scan" (right) -> Scroll to verify "Protect your passwords" and "Protect your location" rows with chevrons | Not Automated |
| **Scan Results — Medium Risk** |  |  |  |  |  |
| 22 | [C2946073](https://testrail.eero.amazon.dev/index.php?/cases/view/2946073) | MED-01 | Medium risk results UI | Login as non-subscriber owner -> Complete scan (medium risk) -> Verify timeline -> Verify yellow/orange banner "Risk summary: Medium" -> Verify upsell card present -> Verify "Free protection" section | Candidate |
| **Scan Results — No Risk** |  |  |  |  |  |
| 23 | [C2946074](https://testrail.eero.amazon.dev/index.php?/cases/view/2946074) | NORISK-01 | No risk results UI | Login as non-subscriber owner -> Complete scan (no risk) -> Verify timeline: only green "Scanned" (no exposure rows) -> Verify green banner "Risk summary: No risk" -> Verify upsell card still present -> Verify "Free protection" section | Candidate |
| **Scan Results — Protected (Plus Subscriber)** |  |  |  |  |  |
| 24 | [C2946075](https://testrail.eero.amazon.dev/index.php?/cases/view/2946075) | PROT-01 | Protected results UI (Plus subscriber) | Login as Plus subscriber owner -> Complete scan (with exposures) -> Verify timeline shows exposures -> Verify blue banner "Risk summary: Protected" -> Verify NO upsell card -> Verify "Protect your passwords" and "Protect your location" rows with eero+ badges -> Verify "Free protection" articles | Candidate |
| 25 | [C2946076](https://testrail.eero.amazon.dev/index.php?/cases/view/2946076) | PROT-02 | Tap Plus action rows | Login as Plus subscriber -> Complete scan -> Tap "Protect your passwords" -> Verify navigates to 1Password -> Navigate back -> Tap "Protect your location" -> Verify navigates to Guardian VPN | Not Automated |
| **Breaches List & Detail** |  |  |  |  |  |
| 26 | [C2946095](https://testrail.eero.amazon.dev/index.php?/cases/view/2946095) | BRE-01 | Breaches list filtered by category | Login as owner -> Complete scan (high risk) -> Tap on specific exposed category in timeline -> Verify breaches list shows only breaches for that category -> Tap individual breach -> Verify detail screen with title, description, site, dates | Candidate |
| 27 | [C2946096](https://testrail.eero.amazon.dev/index.php?/cases/view/2946096) | BRE-02 | Breaches list full (View all) and detail | Login as owner -> Complete scan -> Tap "View all" -> Verify all breaches displayed -> Scroll to verify list is scrollable -> Tap a breach -> Verify detail screen loads -> Tap "Back" -> Verify returns to list -> Tap "Back" -> Verify returns to results | Candidate |
| 28 | [C2946097](https://testrail.eero.amazon.dev/index.php?/cases/view/2946097) | BRE-03 | Breach detail upsell CTA (non-subscriber) | Login as non-subscriber -> Complete scan -> Navigate to breach detail -> Verify "Get eero Plus" CTA visible -> Tap it -> Verify Plus upsell opens | Not Automated |
| 29 | [C2946098](https://testrail.eero.amazon.dev/index.php?/cases/view/2946098) | BRE-04 | Breach detail action CTA (subscriber) | Login as Plus subscriber -> Complete scan -> Navigate to breach detail -> Verify "Set it up now" CTA visible -> Tap it -> Verify navigates to Identity Protection | Not Automated |
| **Troubleshooting Entry Point** |  |  |  |  |  |
| 30 | [C2946099](https://testrail.eero.amazon.dev/index.php?/cases/view/2946099) | TRB-01 | DFP accessible from Troubleshooting | Login as owner -> Navigate to Settings > Troubleshooting -> Verify "Digital footprint scan" row under "Tools & resources" -> Tap row -> Verify DFP bottom sheet opens with email prefilled | Candidate |
| **New Scan / Re-scan** |  |  |  |  |  |
| 31 | [C2946100](https://testrail.eero.amazon.dev/index.php?/cases/view/2946100) | RESCAN-01 | New scan from results screen | Login as owner -> Complete first scan -> On results screen tap "New scan" (right nav) -> Verify bottom sheet opens with email prefilled -> Complete new scan -> Verify results update | Candidate |
| **Analytics** |  |  |  |  |  |
| 32 | [C2946101](https://testrail.eero.amazon.dev/index.php?/cases/view/2946101) | ANA-01 | Analytics events fire through full flow | Login as owner -> Verify home card impression event -> Tap card (verify tap event) -> Complete scan (verify scan initiated + scan completed events) -> Verify upsell shown event (if non-subscriber) -> Close flow | Not Automated |
| 33 | [C2946102](https://testrail.eero.amazon.dev/index.php?/cases/view/2946102) | ANA-02 | Troubleshooting entry point analytics | Login as owner -> Navigate to Troubleshooting -> Tap DFP row -> Verify troubleshooting entry point event fires | Not Automated |
| **Edge Cases** |  |  |  |  |  |
| 34 | [C2946103](https://testrail.eero.amazon.dev/index.php?/cases/view/2946103) | EDGE-01 | Network error during scan | Login as owner -> Tap DFP card -> Enable airplane mode -> Tap "Scan now" -> Verify appropriate error message -> Disable airplane mode -> Retry -> Verify scan completes | Not Automated |
| 35 | [C2946104](https://testrail.eero.amazon.dev/index.php?/cases/view/2946104) | EDGE-02 | App backgrounded during scan | Login as owner -> Start scan -> Background app during "Scanning..." state -> Return to app -> Verify results displayed correctly | Not Automated |
| 36 | [C2946105](https://testrail.eero.amazon.dev/index.php?/cases/view/2946105) | EDGE-03 | Account switch resets card state | Login as owner -> Dismiss DFP card -> Logout -> Login as different owner account -> Verify DFP card appears (fresh state for new account) | Not Automated |
| 37 | [C2946106](https://testrail.eero.amazon.dev/index.php?/cases/view/2946106) | EDGE-04 | Multiple networks permission check | Login as owner of multiple networks -> Verify DFP visible -> Switch to network where user is invited (non-owner) -> Verify DFP not visible -> Switch back -> Verify DFP visible again | Not Automated |
| 38 | [C2946260](https://testrail.eero.amazon.dev/index.php?/cases/view/2946260) | EDGE-05 | Phone-only account (no email) — DFP behavior | Login as owner with phone-only account (no email) -> Navigate to Home -> Verify DFP card behavior -> Navigate to Troubleshooting -> Verify DFP row behavior -> If accessible, verify bottom sheet requires manual email entry | Not Automated |
| **Accessibility** |  |  |  |  |  |
| 39 | [C2946107](https://testrail.eero.amazon.dev/index.php?/cases/view/2946107) | A11Y-01 | VoiceOver/TalkBack full flow | Login as owner with VoiceOver (iOS) or TalkBack (Android) enabled -> Navigate to DFP card -> Verify card is announced -> Activate card -> Verify bottom sheet elements are labeled -> Complete scan -> Verify results screen elements are properly announced | Not Automated |


---

## 7\. Risks

| \# | Risk | Impact | Likelihood | Mitigation |
| :---- | :---- | :---- | :---- | :---- |
| 1 | MWB API instability in production | Scan failures on device | Medium | Mock responses in staging; retry logic in app; monitor MWB uptime |
| 2 | Email verification edge cases (multiple providers, slow networks) | Users unable to complete scan | Medium | Test multiple email providers and network conditions |
| 3 | Feature flag race conditions during active session | Partial UI display | Low | Test flag toggling mid-session on both platforms |
| 4 | 30-day card reappearance difficult to test naturally | Missed regression | Medium | Device clock manipulation + local storage inspection |
| 5 | IAP sandbox inconsistency (Apple/Google) | Upsell flow failures in testing | Medium | Test in Apple/Google sandbox environments early |
| 6 | OS version fragmentation | UI/behavior differences across versions | Medium | Test on min/max supported OS per platform |
| 7 | Localization of scan results (cloud-returned strings) | Truncated/missing translations | Low | Verify all supported languages on both platforms |
| 8 | Mock endpoints not ready for automation | Scan result tests blocked | High | Coordinate with Jason — 4 automated tests depend on mock availability |
| 9 | Risk level determination logic unclear | Wrong banner color/icon displayed | Medium | Clarify with Cloud team which categories map to High vs Medium risk |
| 10 | Existing upsell test overlap | Duplicate test maintenance | Low | DFP tests verify upsell opens but do NOT re-validate upsell modal internals (covered by `test_c967958`) |
| 11 | Custom email verification uses different system | Cannot automate custom email scan flow | Low | VER-01/02/03 remain manual; account email flow (SCAN-01, SCAN-04) is automatable via admin API |
| 12 | Phone-only accounts — unknown DFP behavior | Potential crash or broken UI | Medium | Manual test (EDGE-05) once Jason provides phone-only test account |

---

## 8\. Sign-Off

| Team | DRI (person signing off) | Approval Status |
| :---- | :---- | :---- |
| QA | QA Lead |  |
| QA | Internal QA Reviewer |  |
| SDM | SDM or Project Lead |  |
| Product | Product Manager |  |

**Note:** "Informed sign off" means the DRI from the team (or someone to whom they have delegated) should sign off to acknowledge that they have read the document and had the opportunity to share their questions and concerns.

---

*End of Test Plan*

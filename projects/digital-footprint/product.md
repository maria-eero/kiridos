# Product Requirements

## Priority Definitions

| Priority | Definition |
|----------|-----------|
| P0 | **Ship blocker:** Must be met to launch |
| P1 | **Must do:** Highly desired for day-zero, willing to wait 3-6 months |
| P2 | **Should do:** Nice to have, stretch goals |

## Mobile App Requirements (iOS & Android)

### 1.1 Initiate Scan (P0)

- Users can initiate a scan regardless of subscription status (all residential users across retail and ISP)
- Scan field prefills with account email; user can change to another email
- 2FA email verification to confirm user has access to the email
- Splash screen explaining scan is in progress and what it includes
- User can re-initiate scans with no limit
- Once scan results are available, user can subscribe to Plus or see details
- Plus upsell goes straight from scan details to IAP purchase flow
- Note: EB (eero Business) users do not have access

### 1.2 Scan Entry Points (P0)

- Plus dashboard (if already a Plus subscriber)
- Notification of new data breach changes (Plus subscribers, background scan alerts)
- Card on home screen (if not a Plus subscriber)
- Email messaging or marketing on eero.com/account.eero
- Co-branded or ISP-led marketing campaigns
- Troubleshooting section in Settings

### 1.3 Scan Results (P0)

Categories of exposed information:
- **Passwords** — exposed or leaked
- **Social media & email accounts** — account info exposed online
- **Phone number** — exposed on dark web
- **SSN risk** — identity exposure
- **Location history** — exposed location data
- **Enterprise data breaches** — e.g., ATT or VZW hacks
- **Financial information** — financial data exposure
- **IP addresses** — IP address exposure

If a user has nothing in a category, call out that they've done a great job protecting their identity.

### 1.4 Scan Recommendations & Upsell (P0)

| Exposure Type | Recommendation |
|---------------|---------------|
| Password | eero Plus + 1Password |
| Social media/email | Plus + 1Password |
| Phone number | Plus + Identity Protection |
| SSN | Plus + Identity Protection |
| Location history | Plus + VPN |
| Enterprise data breach | Plus + Identity Protection |

## Mobile App Screens

1. **Home Card Entry Point** — Promo card above Home cards section
2. **Scan Initiation Sheet** — Email input + verification flow
3. **Scan Results Screen** — Categories, breach count, upsell CTA
4. **Breaches List** — Filterable by category or all breaches
5. **Breach Detail** — Individual breach information with action CTAs
6. **Troubleshooting Entry Point** — Alternative entry in Settings

## Home Card Visibility Rules

Display only when ALL conditions are met:
- `feature_132 == true`
- `features.digital_footprint_scan.capability.capable`
- `permissions.digital_footprint_scan.create == true`
- User hasn't dismissed the card (reappears after 30 days)

## Web Payments (account.eero.com) (P0)

- Plus users can access MWB portal from account.eero
- Mobile device visitors are deeplinked to scan flow in eero app
- MWB portal and recurring scans should be co-branded for Plus subscribers

## Marketing (P0)

- Add digital footprint as a Plus feature on eero.com and Plus email campaigns
- Add to a.com Plus SPP and eero Plus bundled ASINs

## Analytics

Key metrics to track:
- How many users perform digital footprint scans?
- How many users purchase eero Plus after performing scans?
- How many users perform a scan from each entry point?

## Open Questions (from Mobile Tech Spec)

- Should ISP users without Plus see this? (Layout TBD, possible next milestone)
- What exactly is "high risk"? → Defined as Password, Location, and SSN
- How are subtitles assembled? → Returned from cloud (localization handled server-side)

## Sources

- [PRD](https://docs.google.com/document/d/1UZuaRqTBeqP2i5JQn8BtpNnmhkDCKWdwxEDoeS_liN4/edit)
- [Mobile Tech Spec](https://docs.google.com/document/d/1yZ3M5ze90yTmJ3I15H0F2co84VP6EloHuWBLNR28zrg/edit)
- [Figma Designs](https://www.figma.com/design/fQfUJ3k8r0XJxt0kQDkFzY/Digital-Footprint-Scan?node-id=155-15072&p=f&t=zgycNyayJJ8rsMgW-0)

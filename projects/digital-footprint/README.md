# Digital Footprint Native Scan

## Overview

The Digital Footprint Native Scan project integrates Malwarebytes' dark web scanning capabilities natively into the eero mobile app. It enables eero users to scan their email address for data breaches and compromised information, providing actionable recommendations to protect their identity online.

**Initiative:** [SWS-31406](https://eeroinc.atlassian.net/browse/SWS-31406)  
**Status:** In Progress  
**Priority:** P1 - Critical  
**DRI:** Amanda Correa  

## Goals

1. Enable a scan based on the user's email address (account email or user-provided)
2. Display a list of potential issues where user information is compromised
3. Provide resources and recommendations for remediation

## Value Proposition

- Offers a critical dark web scan feature that competitors already provide
- Addresses ISP partner requests for inclusion in Plus
- Increases user engagement and retention with eero Plus
- Provides a seamless upsell path from scan results to Plus subscription

## Key Links

- **Confluence:** [Digital Footprint Native Scan](https://eeroinc.atlassian.net/wiki/spaces/sws/pages/4875747344/Digital+Footprint+Native+Scan)
- **Figma:** [Digital Footprint Scan Designs](https://www.figma.com/design/fQfUJ3k8r0XJxt0kQDkFzY/Digital-Footprint-Scan?node-id=155-15072&p=f&t=zgycNyayJJ8rsMgW-0)
- **Slack:** #proj-digital-footprint
- **Jira Initiative:** [SWS-31406](https://eeroinc.atlassian.net/browse/SWS-31406)

## Product Tenets

1. Offer a simple way for all users regardless of subscription status to run a scan in the eero app
2. Build the experience natively in the eero app and provide users a way to seamlessly add Plus to fix identity protection breaches
3. Ensure recommendations for users on password management, 1Password with Plus, and identity trail removal

## Target Release

- Release: 26.7 (Production: June 24th, 2026)
- Code freeze: June 10th, 2026
- Feature flag: `feature_132`
- Available in all countries and languages where Plus subscriptions are supported

## Project Documents

| Document | Description |
|----------|-------------|
| [README.md](./README.md) | This file - project overview |
| [Digital Footprint Mobile Test Plan.md](./Digital%20Footprint%20Mobile%20Test%20Plan.md) | Mobile QA test plan |
| [architecture.md](./architecture.md) | Technical architecture and API design |
| [team.md](./team.md) | Team members and roles |
| [product.md](./product.md) | Product requirements and features |
| [processes.md](./processes.md) | Development processes and workflows |
| [tickets.md](./tickets.md) | Jira ticket breakdown |

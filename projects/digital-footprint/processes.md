# Processes

## Release Strategy

- **Target version:** v26.8
- **Code freeze:** July 1st
- **Feature flag:** `feature_132`
- **Gating:** Feature flag + capability API + network permissions (owner only)
- **Rollback:** Feature flag can be disabled to hide the feature

## Development Workflow

### Phase 1: Scoping & Preparation ([SWS-28286](https://eeroinc.atlassian.net/browse/SWS-28286))
- Kickoff meeting
- Setup project communication channels (#proj-digital-footprint)
- LRR (Legal Risk Review) fill out and approval
- Cloud Tech Spec write and review
- Mobile Tech Spec write and review
- Security consult

### Phase 2: Milestone 1 Implementation ([SWS-28293](https://eeroinc.atlassian.net/browse/SWS-28293))
- Cloud API implementation
- iOS implementation (feature flag, home card, scan initiation, results, breaches, upsell, analytics)
- Android implementation (feature flag, capability, APIs, home card, verification, scan screen, analytics)

### Phase 3: Release Preparation ([SWS-28294](https://eeroinc.atlassian.net/browse/SWS-28294))
- Write PRR document
- Internal bug bash
- PRR internal review
- PRR approval
- Update project page
- Launch meeting

### Phase 4: Fast Follow ([SWS-28295](https://eeroinc.atlassian.net/browse/SWS-28295))
- Deprecate feature flags (iOS and Android)
- Review fast follow items

## Key Decisions

| Decision | Impact | Date | Reference |
|----------|--------|------|-----------|
| Remove contextual flags from first phase | Reduced scope | 4/8/26 | [Slack thread](https://eero.slack.com/archives/C08TYQM701W/p1775679385361609?thread_ts=1775666058.463829&cid=C08TYQM701W) |
| Show boolean exposure indicators instead of PII | Security/privacy improvement | 4/28 | Confluence sync notes |
| eero handles email verification (not MWB) | Security ownership | — | Cloud Tech Spec |

## Risks & Dependencies

| Category | Status |
|----------|--------|
| Security Dependency | NOT STARTED |
| Node Dependency | NOT STARTED |
| Legal Dependency | NOT STARTED |
| Other eero Teams | NOT STARTED |
| External Dependency (MWB) | Resolved |
| PRR | NOT STARTED |

## QA Strategy

- Test plan to be written once final Mobile Tech Spec is available
- P0 and P1 test cases to be automated for launch
- Testing begins once nightly builds are available
- Internal bug bash before PRR

## Weekly Syncs

Project syncs are held weekly (documented on Confluence page). Key updates tracked:
- Implementation progress per platform
- Design review status
- Blocker resolution
- Timeline adjustments

## Sources

- [Confluence Project Page](https://eeroinc.atlassian.net/wiki/spaces/sws/pages/4875747344/Digital+Footprint+Native+Scan)
- [SWS-31406](https://eeroinc.atlassian.net/browse/SWS-31406)

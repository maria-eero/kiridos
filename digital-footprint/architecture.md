# Architecture

## High-Level Flow

```
Mobile App → eero Cloud → Identity ABT Service → Malwarebytes API
```

1. Mobile app initiates scan request
2. eero Cloud checks if user's email is verified
3. If unverified, sends verification email; user confirms
4. Cloud forwards request to Identity ABT Service
5. ABT Service calls Malwarebytes POST Start DFP Scan
6. ABT Service polls GET DFP Scan results
7. Response is mapped and returned through Cloud to Mobile

## API Specification

### Primary Endpoint

**`POST /2.2/account/identity_protection/scan`**

Initiates a digital footprint scan. Typical latency: 1-2 seconds (always triggers a live scan via Malwarebytes).

**Request Body:**
```json
{
  "email": "one-off-scan-email@eero.com"
}
```
> Note: Empty body uses account email.

**Response Body (success):**
```json
{
  "scan_result": {
    "exposed_categories": {
      "email": 4,
      "passwords": 5,
      "ip_addresses": 6,
      "phone_number": 7,
      "identity": 0,
      "online_accounts": 4,
      "location": 0,
      "financial_information": 4
    },
    "source_breachs": [
      {
        "title": "string",
        "short_title": "string",
        "description": "string",
        "site": "string",
        "site_description": "string",
        "published_at": "2019-08-24T14:15:22Z",
        "breached_at": "2019-08-24T14:15:22Z",
        "num_records": 0
      }
    ],
    "included_categories": {
      "email": true,
      "passwords": true,
      "ip_addresses": true
    }
  },
  "summary_string": "",
  "help_articles": [
    {
      "title": "More ways to stay protected",
      "subtitle": "View our guide to learn other ways you can protect your information",
      "url": "https://eero.com/help"
    }
  ],
  "identity_protection_enabled": true,
  "mwb_full_report_url": "https://www.malwarebytes.com/digital-footprint"
}
```

**Response Body (email unverified):**
```json
{
  "email_unverified": "true"
}
```

### Email Verification

**`POST /2.2/account/identity_protection/verify_email`**

```json
{
  "code": "123456"
}
```

### Feature Capability

**`GET /2.2/entitlements/networks/:id/features`**

```json
{
  "data": {
    "features": [
      {
        "digital_footprint_scan": {
          "capable": true
        }
      }
    ]
  }
}
```

### Permissions

**`GET /2.2/networks/:id/permissions`**

Only network owners have access:
```json
{
  "permissions": {
    "digital_footprint_scan": {
      "create": true,
      "update": true,
      "edit": true,
      "delete": true
    }
  }
}
```

## Data Categories

The scan reports exposure across 8 categories (no PII is displayed directly):

| Category | Description |
|----------|-------------|
| Email | Email addresses exposed |
| Passwords | Leaked credentials |
| IP Addresses | IP address exposure |
| Phone Number | Phone number on dark web |
| Identity | SSN/identity risk |
| Online Accounts | Social media/email accounts exposed |
| Location | Exposed location history |
| Financial Information | Financial data breaches |

## Feature Gating

The feature requires all three conditions:
- Feature flag: `feature_132 == true`
- Capability: `features.digital_footprint_scan.capability.capable == true`
- Permission: `permissions.digital_footprint_scan.create == true`

## Cloud Workload

- New API client for Malwarebytes DFP capability in Identity Protection service
- Email verification for non-account emails
- Passthrough logic in eero Cloud for endpoints
- Mapping between MWB PII categories and 8 in-app categories
- Max 100 breaches returned per scan

## Security Considerations

- eero handles email verification (not Malwarebytes)
- No PII displayed in plain text — only boolean exposure indicators per category
- Security team focused on how locked down the service handling this data will be
- Threat model required (outside standard ASR process)
- Data classification review needed

## Performance Requirements

- Must not slow down eero boot time
- Must not slow down app load time
- Must not impact network setup time
- Must not increase cloud costs
- Must not reduce wifi performance
- New data should populate in 30 seconds or less

## References

- [Cloud Tech Spec](https://docs.google.com/document/d/1x3twjnG2ugyN1wBOStBr-gFXYb2PhTI57ZOTIj8CMi8/edit)
- [Mobile Tech Spec](https://docs.google.com/document/d/1yZ3M5ze90yTmJ3I15H0F2co84VP6EloHuWBLNR28zrg/edit)
- [Identity Protection Cloud Tech Spec](https://docs.google.com/document/d/1NWKelZTAE8-4HYiOfDL2imLLXT82v59s6VLkxBoe4VY/edit)

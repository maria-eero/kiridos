---
name: testrail-case-creation
description: Use when asked to create test cases in TestRail from a test plan. Generates properly formatted test cases following the established patterns and pushes them via the TestRail API.
---

# TestRail Test Case Creation Skill

Generate and create test cases in TestRail from a test plan document, following the established team patterns for case structure, naming, and metadata.

## Authentication

- **Token**: Stored in `~/.zshrc` as `TESTRAIL_TOKEN` (Base64 encoded email:apikey)
- **Base URL**: `https://testrail.eero.amazon.dev`
- **Auth Header**: `Authorization: Basic $TESTRAIL_TOKEN`

## TestRail Instance Configuration

- **Project ID**: `2` (eero SW QA)
- **Suite ID**: `156` (B2B Mobile)
- **Template ID**: `2` (Test Case - Steps Separated)

## API Reference

### Read Operations
```
GET /api/v2/get_sections/{project_id}&suite_id={suite_id}
GET /api/v2/get_cases/{project_id}&suite_id={suite_id}&section_id={section_id}
GET /api/v2/get_case/{case_id}
```

### Write Operations
```
POST /api/v2/add_section/{project_id}  → {"suite_id": 156, "name": "...", "parent_id": ...}
POST /api/v2/add_case/{section_id}     → {case payload}
POST /api/v2/update_case/{case_id}     → {fields to update}
```

## Test Case Pattern (from existing cases)

Each test case follows this structure:

### Title Convention
- Descriptive, starts with "Verify..." 
- Includes the specific condition being tested
- Example: `Verify Multistatic IP row is hidden for retail networks`
- Example: `Verify users are not able to navigate to the 'Public static IP' screen if they don't have update permissions and the feature is disabled`

### Template: Steps Separated (template_id: 2)

```json
{
  "title": "Verify [what is being verified]",
  "template_id": 2,
  "type_id": 6,
  "priority_id": 4,
  "custom_is_automatable": true,
  "custom_test_case_status": 1,
  "custom_automation_status": 8,
  "custom_automation_type": 7,
  "custom_preconds": "- Precondition 1\n- Precondition 2\n- Precondition 3",
  "custom_steps_separated": [
    {"content": "Step 1 action", "expected": "Step 1 expected result"},
    {"content": "Step 2 action", "expected": "Step 2 expected result"},
    {"content": "Step 3 action", "expected": "Step 3 expected result"}
  ]
}
```

### Preconditions Pattern
Written as a bulleted list (markdown-style with `- `):
```
- Be on a Residential network
- Be logged in as the Residential network owner
- Enable feature flag 'feature_132'
- Network must not be in bridge mode
```

### Steps Pattern
Each step has:
- **content**: The action to perform (imperative, clear)
- **expected**: What should happen after the action

Steps should be:
- Granular enough to follow without ambiguity
- Not so granular that each tap is a separate step
- Navigation steps can be combined: "Log in as owner and tap on 'Settings'"
- Verification steps include the specific UI element/text to check

### Example Case (from Multistatic IP section)
```json
{
  "title": "Verify Multistatic IP row is hidden for retail networks",
  "template_id": 2,
  "type_id": 6,
  "priority_id": 4,
  "custom_is_automatable": true,
  "custom_test_case_status": 1,
  "custom_automation_status": 8,
  "custom_automation_type": 7,
  "custom_preconds": "- Be on a retail network\n- Be logged in as a network owner\n- Enable feature flag 'feature_76'",
  "custom_steps_separated": [
    {
      "content": "Log in as a retail network owner and tap on 'Settings'",
      "expected": "'Settings' screen appears"
    },
    {
      "content": "Tap 'Network settings'",
      "expected": "'Network settings' screen appears and the Multistatic IP row will not be present on this screen"
    }
  ]
}
```

## Field Reference

### Priority (priority_id)
| ID | Name |
|----|------|
| 4 | P0 |
| 3 | P1 |
| 2 | P2 |
| 1 | P3 |
| 5 | P4 (To be reviewed) |

### Type (type_id)
| ID | Name | Use When |
|----|------|----------|
| 6 | Functional | Default for feature test cases |
| 1 | Acceptance | Acceptance criteria validation |
| 9 | Regression | Regression-specific cases |
| 11 | Smoke | Critical path smoke tests |

### Automation Type (custom_automation_type)
| ID | Name |
|----|------|
| 0 | None |
| 7 | Mobile - Appium |
| 3 | Mobile Automation-Sofy |
| 4 | Mobile Automation-Both |

### Automation Status (custom_automation_status)
| ID | Name |
|----|------|
| 1 | Ready for Automation |
| 2 | Automated - iOS |
| 3 | Automated - Android |
| 4 | Automated - Both iOS and Android |
| 8 | Not Automated |

### Test Case Status (custom_test_case_status)
| ID | Name |
|----|------|
| 1 | Draft |
| 2 | TA Ready |
| 3 | Needs Review |
| 4 | Obsolete |
| 5 | Duplicate |

### Org User Role (custom_org_user_role) — multi-select
| ID | Name |
|----|------|
| 1 | Pro Installer |
| 2 | ISP Technician |
| 5 | ISP Admin |
| 10 | Business Owner |
| 16 | Retail network owner |
| 17 | Retail network admin |

## Workflow

### Step 1: Identify or Create Feature Section

Check if a section already exists for the feature:
```bash
curl -s -H "Authorization: Basic $TESTRAIL_TOKEN" \
  "https://testrail.eero.amazon.dev/index.php?/api/v2/get_sections/2&suite_id=156" \
  | python3 -c "import json,sys; [print(f'{s[\"id\"]}: {s[\"name\"]}') for s in json.load(sys.stdin) if 'FEATURE_NAME' in s['name'].lower()]"
```

If not found, create one:
```bash
curl -s -H "Authorization: Basic $TESTRAIL_TOKEN" -H "Content-Type: application/json" \
  -X POST "https://testrail.eero.amazon.dev/index.php?/api/v2/add_section/2" \
  -d '{"suite_id": 156, "name": "Feature Name"}'
```

### Step 2: Create Sub-Sections

Create sub-sections under the feature section matching the test plan categories. Each category from the test plan (Feature Gating, Home Card, Scan Initiation, etc.) becomes a sub-section:

```bash
curl -s -H "Authorization: Basic $TESTRAIL_TOKEN" -H "Content-Type: application/json" \
  -X POST "https://testrail.eero.amazon.dev/index.php?/api/v2/add_section/2" \
  -d '{"suite_id": 156, "name": "Category Name", "parent_id": FEATURE_SECTION_ID}'
```

Sub-sections should mirror the test plan scenario groups exactly. Example structure:
```
Feature Name (parent)
├── Feature Gating
├── Home Card Entry Point
├── Scan Initiation
├── Email Verification
├── Scan Results - High Risk
├── Scan Results - Medium Risk
├── Scan Results - No Risk
├── Scan Results - Protected
├── Breaches List & Detail
├── Troubleshooting Entry Point
├── New Scan
├── Analytics
├── Edge Cases
└── Accessibility
```

### Step 3: Transform Test Plan Scenarios to Cases

For each test scenario in the test plan:
1. Map the scenario title → case title (prefix with "Verify")
2. Extract preconditions from the user type and setup requirements
3. Break the scenario flow into discrete steps with expected results
4. Set priority based on test plan priority (P0 → priority_id 4, P1 → priority_id 3)
5. Set `custom_is_automatable` based on whether the flow can be automated
6. Set `custom_automation_type` to 7 (Mobile - Appium) for automatable cases

### Step 3: Transform Test Plan Scenarios to Cases

For each test scenario in the test plan:
1. Map the scenario title → case title (prefix with "Verify")
2. Extract preconditions from the user type and setup requirements
3. Break the scenario flow into discrete steps with expected results
4. Set priority based on test plan priority (P0 → priority_id 4, P1 → priority_id 3)
5. Set `custom_is_automatable` based on whether the flow can be automated
6. Set `custom_automation_type` to 7 (Mobile - Appium) for automatable cases

### Step 4: Create Cases in Correct Sub-Sections

Create each case directly in its target sub-section:
```bash
curl -s -H "Authorization: Basic $TESTRAIL_TOKEN" -H "Content-Type: application/json" \
  -X POST "https://testrail.eero.amazon.dev/index.php?/api/v2/add_case/{sub_section_id}" \
  -d '{case_payload_json}'
```

If cases were created in the parent section, move them to sub-sections:
```bash
curl -s -H "Authorization: Basic $TESTRAIL_TOKEN" -H "Content-Type: application/json" \
  -X POST "https://testrail.eero.amazon.dev/index.php?/api/v2/move_cases_to_section/{target_section_id}" \
  -d '{"suite_id": 156, "case_ids": [case_id_1, case_id_2]}'
```

### Step 5: Report Results

After creation, output a summary:
```
Created X test cases in TestRail section "Feature Name" (ID: XXXXX)
Suite: B2B Mobile (156)
Link: https://testrail.eero.amazon.dev/index.php?/suites/view/156&group_by=cases:section_id&group_order=asc&display_deleted_cases=0&group_id={section_id}

Cases created:
  C{id}: {title} (P0)
  C{id}: {title} (P1)
  ...
```

## Rules

1. **Always use template_id 2** (Steps Separated) — never use plain text steps
2. **Always organize cases in sub-sections** — create sub-sections matching test plan categories under the feature parent section
3. **Always use template_id 2** (Steps Separated) — never use plain text steps
4. **Always set custom_automation_type to 7** (Mobile - Appium) for automatable mobile cases
5. **Default custom_automation_status to 8** (Not Automated) for new cases
6. **Default custom_test_case_status to 1** (Draft) for new cases
7. **First case in section should be a Template case** showing the preconditions pattern for the section
8. **Preconditions must include**: network type, user role, feature flag, any special setup, and Figma link where applicable
9. **Steps should match the test plan flow**: Login → Navigate → Verify (same granularity as the plan)
10. **Do NOT create cases that duplicate existing coverage** — check existing sections first
11. **Ask user to confirm** the list of cases before creating them in TestRail
12. **Create cases directly in sub-sections** — use the sub-section ID in the add_case endpoint, not the parent section

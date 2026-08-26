# HiBob Employee Lifecycle

Employee lifecycle templates for HiBob, Jira Service Management, Jira work items, and Microsoft Entra.

The template installs three flows:

- `HiBob - Employee Onboarding`: receives a new-employee webhook, creates a JSM onboarding request, creates IT tasks, creates the Entra user, adds group memberships, and optionally assigns license SKUs.
- `HiBob - Employee Offboarding`: receives a termination webhook, creates a JSM offboarding workflow request, disables the Entra user, removes configured groups/licenses, creates equipment-return and access-verification tasks, and creates a final completion task.
- `HiBob - Employee Mover`: receives department, manager, role, or location changes, creates a mover request, updates Entra profile/manager data, adjusts configured groups, and creates access, hardware, and software review tasks.

## Required Customer Configuration

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `{tenant}` | `manifest.json` Entra OAuth2 URLs | Replace with the customer's Microsoft Entra tenant ID. |
| `example.com` | `ENTRA_DOMAIN` flow variable | Replace with the customer's fallback Entra UPN domain. |
| `ChangeMe-Replace-123!` | `ENTRA_TEMP_PASSWORD` flow variable | Replace with the customer's temporary password policy value or change the user-create action. |
| `00000000-0000-0000-0000-000000000000` | `ENTRA_ONBOARDING_GROUP_ID` | Replace with the baseline onboarding/access group object ID. |
| `[]` | `ENTRA_ADDITIONAL_GROUP_IDS`, `ENTRA_GROUP_IDS_TO_ADD`, `ENTRA_GROUP_IDS_TO_REMOVE`, license SKU variables | Replace with JSON arrays or comma-separated values when group/license changes should be configured in the flow instead of supplied by the webhook payload. |
| `1`, `100`, `101`, `102` | JSM service desk/request type variables | Replace with the customer's service desk ID and onboarding/offboarding/mover request type IDs. |
| `customfield_10153`, `customfield_10154`, `customfield_10155`, `customfield_10156` | JSM request fields | Replace with the customer's Jira custom field IDs. |
| `HR` | `TASK_PROJECT` variable | Replace with the target Jira project key. |
| `Sub-task` | `TASK_ISSUE_TYPE` variable | Replace with `Task` if the target project does not support subtasks. |

## Webhook Payloads

Configure HiBob webhooks, reports, integrations, or middleware to post to the iHub custom webhook URL for the relevant flow. The flows accept common flat fields and nested variants under `employee.*`, `worker.*`, `data.employee.*`, `employment.*`, `jobInfo.*`, `work.*`, and `attributes.*.value`.

Recommended new-employee payload:

```json
{
  "eventType": "employee.created",
  "employeeId": "100123",
  "firstName": "Avery",
  "lastName": "Stone",
  "displayName": "Avery Stone",
  "email": "avery.stone@example.com",
  "startDate": "2026-09-01",
  "managerId": "100001",
  "department": "Finance",
  "location": "Stockholm",
  "jobTitle": "Business Controller",
  "groupIdsToAdd": ["00000000-0000-0000-0000-000000000000"],
  "licenseSkuIdsToAssign": ["11111111-1111-1111-1111-111111111111"]
}
```

Recommended termination payload:

```json
{
  "eventType": "employee.terminated",
  "employeeId": "100123",
  "displayName": "Avery Stone",
  "email": "avery.stone@example.com",
  "endDate": "2026-12-31",
  "groupIdsToRemove": ["00000000-0000-0000-0000-000000000000"],
  "licenseSkuIdsToRemove": ["11111111-1111-1111-1111-111111111111"]
}
```

Recommended mover payload:

```json
{
  "eventType": "employee.moved",
  "employeeId": "100123",
  "displayName": "Avery Stone",
  "email": "avery.stone@example.com",
  "effectiveDate": "2026-10-01",
  "previousDepartment": "Finance",
  "department": "Operations",
  "previousLocation": "Stockholm",
  "location": "London",
  "previousManagerId": "100001",
  "managerId": "100045",
  "managerEntraUserId": "manager@example.com",
  "groupIdsToRemove": ["22222222-2222-2222-2222-222222222222"],
  "groupIdsToAdd": ["33333333-3333-3333-3333-333333333333"]
}
```

## Credentials

- `oauth2-hibob-entra`: Microsoft Entra OAuth2 credential. The template requests `offline_access User.Read User.ReadWrite.All GroupMember.ReadWrite.All LicenseAssignment.ReadWrite.All` for user creation/update, group membership changes, and license assignment/removal.

## Notes

- All flows are disabled by default.
- The onboarding flow falls back to `{{employeeId}}@{{_flow.ENTRA_DOMAIN}}` when the HR payload has no email address. Offboarding and mover Entra actions require an email/UPN.
- The group and license actions use Microsoft Graph object IDs/SKU IDs. If HR events do not carry those IDs, configure the corresponding flow variables before enabling the flows.

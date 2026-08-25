# SAP SuccessFactors

Employee lifecycle templates for SAP SuccessFactors, Jira Service Management, Jira Assets, and Microsoft Entra.

The template installs four flows:

- `SAP SuccessFactors - Employee Created`: receives a SuccessFactors employee-created webhook, creates a JSM onboarding request, creates onboarding tasks, creates the Entra user, and adds the user to a baseline Entra group.
- `SAP SuccessFactors - Employee Updated`: receives an employee-updated webhook, patches selected Entra profile fields, and comments on the linked Jira issue.
- `SAP SuccessFactors - Employee Terminated`: receives an employee-terminated webhook, creates a JSM offboarding request, disables the Entra user, and creates recovery/revocation tasks.
- `SAP SuccessFactors - Sync Employee to Jira Assets`: scheduled import from SuccessFactors OData v2 into Jira Assets.

## Required Customer Configuration

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `{tenant}` | `manifest.json` Entra OAuth2 URLs | Replace with the customer's Microsoft Entra tenant ID. |
| `example.com` | `ENTRA_DOMAIN` flow variable | Replace with the customer's fallback Entra UPN domain. |
| `ChangeMe-Replace-123!` | `ENTRA_TEMP_PASSWORD` flow variable | Replace with the customer's temporary password policy value or change the user-create action. |
| `00000000-0000-0000-0000-000000000000` | `ENTRA_ONBOARDING_GROUP_ID`, `importSourceId` | Replace with the Entra group object ID and Jira Assets import source ID. |
| `1`, `100`, `101` | JSM service desk/request type variables | Replace with the customer's service desk ID and onboarding/offboarding request type IDs. |
| `customfield_10153`, `customfield_10154`, `customfield_10155` | JSM request fields and Jira lookup fields | Replace with the customer's Jira custom field IDs. Also replace `cf[10153]` in JQL. |
| `HR` | `SPACE` / `TASK_PROJECT` variables | Replace with the target Jira project key. |
| `Sub-task` | `TASK_ISSUE_TYPE` variables | Replace with `Task` if the target project does not support subtasks. |
| `https://api.successfactors.com` | `SUCCESSFACTORS_URL` | Replace with the customer's SuccessFactors API base URL. |
| SuccessFactors OData `$select` / `$expand` fields | `successfactors_employee_asset_import.json` | Align with the entities and permissions enabled in the customer's tenant. |

## SuccessFactors Webhook Payloads

Configure SuccessFactors Integration Center, Intelligent Services, or middleware to post to the iHub custom webhook URL for the relevant flow.

Recommended employee-created payload:

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
  "jobTitle": "Business Controller"
}
```

Recommended employee-updated payload:

```json
{
  "eventType": "employee.updated",
  "employeeId": "100123",
  "displayName": "Avery Stone",
  "email": "avery.stone@example.com",
  "department": "Finance",
  "location": "Stockholm",
  "jobTitle": "Senior Business Controller"
}
```

Recommended employee-terminated payload:

```json
{
  "eventType": "employee.terminated",
  "employeeId": "100123",
  "displayName": "Avery Stone",
  "email": "avery.stone@example.com",
  "endDate": "2026-12-31"
}
```

The webhook flows also accept common variants such as `personIdExternal`, `userId`, nested `employee.*`, `employment.*`, `jobInfo.*`, and `userNav.*`.

## Credentials

- `basic-auth-sap-successfactors`: SuccessFactors OData credential for employee asset import.
- `oauth2-sap-successfactors-entra`: Microsoft Entra OAuth2 credential. The template requests `offline_access User.Read User.ReadWrite.All Group.ReadWrite.All`.
- `token-sap-successfactors-employee-asset-import-token`: Jira Assets external import token.

## Notes

- All flows are disabled by default.
- The Entra actions assume the SuccessFactors event includes an email/UPN. If not, `Employee Created` falls back to `{{employeeId}}@{{_flow.ENTRA_DOMAIN}}`; the update and terminate flows skip Entra changes when no email is present.
- The Jira Assets import is intentionally single-page (`PAGE_SIZE`). Add pagination if the customer's employee count exceeds the configured limit.

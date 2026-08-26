# SAP SuccessFactors

Employee lifecycle templates for SAP SuccessFactors, Jira Service Management, and Jira Assets.

The template installs four flows:

- `SAP SuccessFactors - Employee Created`: receives a SuccessFactors employee-created webhook, creates a JSM onboarding request, and creates onboarding tasks for identity, hardware, and software provisioning.
- `SAP SuccessFactors - Employee Updated`: receives an employee-updated webhook, creates a JSM mover request, and creates access, hardware/location, and software review tasks.
- `SAP SuccessFactors - Employee Terminated`: receives an employee-terminated webhook, creates a JSM offboarding request, and creates recovery/revocation tasks.
- `SAP SuccessFactors - Sync Employee to Jira Assets`: scheduled import from SuccessFactors OData v2 into Jira Assets.

## Required Customer Configuration

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `00000000-0000-0000-0000-000000000000` | `importSourceId` | Replace with the Jira Assets import source ID. |
| `1`, `100`, `101`, `102` | JSM service desk/request type variables | Replace with the customer's service desk ID and onboarding/offboarding/mover request type IDs. |
| `customfield_10153`, `customfield_10154`, `customfield_10155`, `customfield_10156` | JSM request fields and Jira lookup fields | Replace with the customer's Jira custom field IDs. Also replace `cf[10153]` in JQL. |
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

Recommended employee-updated (mover) payload:

```json
{
  "eventType": "employee.updated",
  "employeeId": "100123",
  "displayName": "Avery Stone",
  "effectiveDate": "2026-10-01",
  "previousDepartment": "Finance",
  "department": "Operations",
  "previousLocation": "Stockholm",
  "location": "London",
  "previousManagerId": "100001",
  "managerId": "100045",
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
- `token-sap-successfactors-employee-asset-import-token`: Jira Assets external import token.

## Notes

- All flows are disabled by default.
- The Jira Assets import is intentionally single-page (`PAGE_SIZE`). Add pagination if the customer's employee count exceeds the configured limit.

# Factorial Employee Lifecycle

Employee lifecycle templates for Factorial, Jira Service Management, and Jira work items.

The template installs three flows:

- `Factorial - Employee Onboarding`: receives a new-employee webhook, creates a JSM onboarding request, and creates IT tasks for identity setup, hardware provisioning, and software provisioning.
- `Factorial - Employee Offboarding`: receives a termination webhook, creates a JSM offboarding workflow request, and creates equipment-return, access-verification, and completion tasks.
- `Factorial - Employee Mover`: receives department, manager, role, or location changes, creates a mover request, and creates access, hardware, and software review tasks.

## Required Customer Configuration

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `1`, `100`, `101`, `102` | JSM service desk/request type variables | Replace with the customer's service desk ID and onboarding/offboarding/mover request type IDs. |
| `customfield_10153`, `customfield_10154`, `customfield_10155`, `customfield_10156` | JSM request fields | Replace with the customer's Jira custom field IDs. |
| `HR` | `TASK_PROJECT` variable | Replace with the target Jira project key. |
| `Sub-task` | `TASK_ISSUE_TYPE` variable | Replace with `Task` if the target project does not support subtasks. |

## Webhook Payloads

Configure Factorial webhooks, reports, integrations, or middleware to post to the iHub custom webhook URL for the relevant flow. The flows accept common flat fields and nested variants under `employee.*`, `worker.*`, `data.employee.*`, `employment.*`, `jobInfo.*`, `work.*`, and `attributes.*.value`.

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
  "jobTitle": "Business Controller"
}
```

Recommended termination payload:

```json
{
  "eventType": "employee.terminated",
  "employeeId": "100123",
  "displayName": "Avery Stone",
  "email": "avery.stone@example.com",
  "endDate": "2026-12-31"
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
  "managerId": "100045"
}
```

## Notes

- All flows are disabled by default.

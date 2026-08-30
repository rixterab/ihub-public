# Dynamics 365 - Create Case from Jira

This template installs one iHub flow that creates a Microsoft Dynamics 365 case (`incident`) whenever a work item is created in a configured Jira project.

- `Dynamics 365 - Create Case from Jira`: Jira -> Dynamics 365, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from Dynamics into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in the `SPACE` flow variable.
- Ignores work items that already carry a Dynamics case ID, so a redelivered webhook never creates a second case.
- Creates one `incident` record in the configured Dynamics environment.
- Writes the created case GUID back to a Jira custom field, which is what makes the previous step work.

## Dynamics API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Dynamics 365 Case | `POST` | `{{_flow.DYNAMICS_365_URL}}/api/data/v9.2/incidents` |
| Store Dynamics Case ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` |

The create action sends `Prefer: return=representation` so Dataverse returns the created row instead of an empty `204`, and stores `$.response.data.incidentid` in `dynamicsCaseId`. Without that header there is no ID to write back.

The body sets only `title` and `description`. `ticketnumber` is a system-maintained autonumber column in Dataverse and is not written by this template.

## Credential

The manifest defines one credential: `token-dynamics-365`, of type `TOKEN`.

Dataverse authenticates with a Microsoft Entra ID OAuth2 access token whose audience is the environment URL. Register an application in Entra ID, grant it the `Dynamics CRM user_impersonation` permission, create an application user for it in the Dynamics environment under `Settings` -> `Security` -> `Application users`, and give that user a security role that can create cases.

The `TOKEN` credential type sends the value as `Authorization: Bearer <token>`, so paste the access token alone. **Entra access tokens expire, typically within an hour.** For anything beyond a proof of concept, replace this credential with an OAuth2 credential that iHub can refresh, rather than pasting a token that will stop working the same day.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Dynamics case GUID. The template uses `customfield_10153`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `DYNAMICS` | `SPACE` flow variable | Replace with the Jira project key that should create cases. |
| `https://your-org.crm.dynamics.com` | `DYNAMICS_365_URL` flow variable | Replace with your Dynamics environment URL. |
| `customfield_10153` | Flow JSON, two places | Replace with the customer's Jira custom field key. |
| Empty token | `token-dynamics-365` credential | Set to a Dataverse access token, or swap in a refreshable OAuth2 credential. |

### Places To Update The Jira Custom Field

- `Create Dynamics 365 Case` condition: checks `$.issue.fields.customfield_10153` is empty before creating.
- `Store Dynamics Case ID on Jira Issue` body: writes the created GUID into `customfield_10153`.

## Limitations

- One-way only. Dynamics case changes are not written back to Jira. The ID write-back here is the link an incoming flow would search on.
- Create only. Later Jira updates and comments are not synced.
- No customer lookup. A real Dynamics case usually needs `customerid_account` or `customerid_contact` set; this template does not resolve a Jira reporter to a Dynamics account or contact. If your environment makes the customer field required, the create call fails until you add that lookup.
- Adjust field names if the environment uses custom columns or has additional required fields.
- The icon is the generic Microsoft logo rather than the Dynamics 365 product mark.

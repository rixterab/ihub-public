# HubSpot - Create Ticket from Jira

This template installs one iHub flow that creates a HubSpot ticket whenever a work item is created in a configured Jira project.

- `HubSpot - Create Ticket from Jira`: Jira -> HubSpot, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from HubSpot into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in the `SPACE` flow variable.
- Ignores work items that already carry a HubSpot ticket ID, so a redelivered webhook never creates a second ticket.
- Creates one HubSpot ticket in the configured pipeline and stage.
- Writes the created ticket ID back to a Jira custom field, which is what makes the previous step work.

The ticket `content` carries a link back to the Jira work item plus its priority and reporter.

## HubSpot API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Create HubSpot Ticket | `POST` | `https://api.hubapi.com/crm/v3/objects/tickets` |
| Store HubSpot Ticket ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` |

The create action stores `$.response.data.id` in `hubspotTicketId`.

`hs_pipeline` and `hs_pipeline_stage` are required and must be **IDs**, not display names. Read them from `GET https://api.hubapi.com/crm/v3/pipelines/tickets` with the same token. The defaults in the template (`0` and `1`) are the IDs of the default HubSpot ticket pipeline and its first stage on many portals, but they are not guaranteed and must be verified.

## Credential

The manifest defines one credential: `token-hubspot`, of type `TOKEN`.

Create a private app in HubSpot under `Settings` -> `Integrations` -> `Private apps`, grant it the `tickets` write scope, and paste the access token (it starts with `pat-`). The `TOKEN` credential type sends it as `Authorization: Bearer <token>`, so paste the token alone without the `Bearer` prefix.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the HubSpot ticket ID. The template uses `customfield_10153`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `HUBSPOT` | `SPACE` flow variable | Replace with the Jira project key that should create HubSpot tickets. |
| `0` | `HUBSPOT_PIPELINE_ID` flow variable | Replace with the target ticket pipeline ID from the pipelines API. |
| `1` | `HUBSPOT_STAGE_ID` flow variable | Replace with the target stage ID from the pipelines API. |
| `customfield_10153` | Flow JSON, two places | Replace with the customer's Jira custom field key. |
| Empty token | `token-hubspot` credential | Set to the private app access token. |

### Places To Update The Jira Custom Field

- `Create HubSpot Ticket` condition: checks `$.issue.fields.customfield_10153` is empty before creating.
- `Store HubSpot Ticket ID on Jira Issue` body: writes the created ID into `customfield_10153`.

## Limitations

- One-way only. HubSpot ticket changes are not written back to Jira. The other direction needs a second flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a HubSpot webhook subscription; the ID write-back here is the link it would search on.
- Create only. Later Jira updates and comments are not synced.
- No associations. The ticket is not linked to a HubSpot contact, company or deal. Add an association action against `/crm/v4/objects/tickets/{id}/associations/...` if the ticket must sit under a customer record.
- The Jira description is not sent. Jira descriptions are ADF and HubSpot ticket `content` is plain text; the link back to Jira is the right affordance here.

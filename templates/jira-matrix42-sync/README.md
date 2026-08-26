# Jira <-> Matrix42 Ticket Sync

This template installs two iHub flows that keep Jira issues and Matrix42 tickets linked in both directions. iHub is the master of the integration: it listens to Matrix42 workflow events and to Jira webhooks, and drives every create and update on both sides.

- `Matrix42 Incoming Sync`: Matrix42 -> Jira, triggered by a Matrix42 Workflow (Process Designer) HTTP activity posting to an iHub custom webhook URL.
- `Matrix42 Outgoing Sync`: Jira -> Matrix42, triggered by Jira webhooks.

The sync stores the Matrix42 ticket `Id` (GUID) in a Jira custom field. That field is the link key used by both flows to find the matching record.

## What The Template Syncs

From Matrix42 to Jira:

- Searches Jira for an issue whose Matrix42 ticket id custom field matches the incoming ticket.
- Creates a Jira issue when Matrix42 sends a `ticket.created` event and no matching Jira issue exists.
- Updates the matching Jira issue summary and description on a `ticket.updated` event.
- Adds Matrix42 notes to the matching Jira issue.
- Downloads Matrix42 attachments by attachment id and uploads them to the matching Jira issue.
- Stores incoming Matrix42 note and attachment ids as Jira issue properties using keys like `ihub-<id>`.

From Jira to Matrix42:

- Creates a Matrix42 ticket when a Jira issue is created and the Jira issue does not already have a Matrix42 ticket id.
- Stores the created Matrix42 ticket id back on the Jira issue.
- Updates the Matrix42 ticket `Subject` and `Description` when the Jira issue is updated.
- Adds Jira comments to the linked Matrix42 ticket as notes.
- Uploads Jira attachments to the linked Matrix42 ticket.
- Stores synced comment and attachment ids as Jira issue properties using keys like `ihub-<id>` to avoid duplicate processing.

## Matrix42 API Notes

Matrix42 exposes its data through an OData REST API whose base path, entity set names and field casing vary by tenant, module version and customization (on-prem vs. cloud). This template assumes an OData v3/v4-style endpoint and generic entity names (`Tickets`, `Notes`, `Attachments`) as a starting point — **before enabling either flow, check the tenant's `$metadata` document** (`{{M42_URL}}/{{M42_API_PATH}}/$metadata`) and confirm the actual entity set name, property names (`Subject`/`Description`/`Id`/`DisplayName`), and note/attachment entity structure, then adjust the flow variables and action bodies to match.

| Action | Method | Endpoint |
| --- | --- | --- |
| Create ticket | `POST` | `{{_flow.M42_URL}}/{{_flow.M42_API_PATH}}/{{_flow.M42_ENTITY}}` |
| Update ticket | `PATCH` | `{{_flow.M42_URL}}/{{_flow.M42_API_PATH}}/{{_flow.M42_ENTITY}}({{issue.fields.customfield_10153}})` |
| Add note | `POST` | `{{_flow.M42_URL}}/{{_flow.M42_API_PATH}}/Notes` |
| Upload attachment | `POST` | `{{_flow.M42_URL}}/{{_flow.M42_API_PATH}}/Attachments` |
| Download attachment | `GET` | `{{_flow.M42_URL}}/{{_flow.M42_API_PATH}}/Attachments({{matrix42AttachmentId}})/$value` |

The template authenticates with Basic Auth using a Matrix42 integration/service account. If the tenant requires OAuth2 instead, replace the `basic-auth-matrix42` credential in `manifest.json` and both flow JSON files with an OAuth2 credential definition and point it at the tenant's token endpoint.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Matrix42 ticket `Id`. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://your-instance.matrix42.cloud` | Incoming and outgoing `M42_URL` flow variable | Replace with the customer's Matrix42 instance URL. |
| `M42Services/api/odata/v3` | Incoming and outgoing `M42_API_PATH` flow variable | Replace with the confirmed OData API path from the tenant's `$metadata` document. |
| `Tickets` | Outgoing `M42_ENTITY` flow variable | Replace with the entity set holding the tickets, for example `Incidents` or `SDIssues`. |
| `M42` | Incoming `SPACE` flow variable | Replace with the Jira project key where Matrix42-created tickets should create/find issues. |
| `Task` | Incoming `ISSUE_TYPE` flow variable | Replace if the target Jira project does not use `Task`. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular comment and attachment sync. |
| `integration@company` | `manifest.json` credential definition | Replace with the customer's Matrix42 integration account username before importing, or edit the credential after import. |

## Matrix42 Workflow Setup

After the template is instantiated in iHub, copy the trigger URL from the `Matrix42 Incoming Sync` custom webhook trigger.

In Matrix42:

1. Open Process Designer (or the equivalent Workflow module for the installed Matrix42 version).
2. Create a workflow triggered on ticket create, update, note-added and attachment-added events for the relevant ticket type.
3. Add an HTTP/REST activity that POSTs a JSON payload to the iHub trigger URL.
4. Map the payload fields to the contract below.

Recommended payload for a ticket create event:

```json
{
  "eventType": "ticket.created",
  "ticketId": "3f1c9a12-4e6a-4b0c-9d21-1a2b3c4d5e6f",
  "displayName": "TCK-1042",
  "subject": "Ticket subject",
  "description": "Ticket description"
}
```

Recommended payload for a ticket update event:

```json
{
  "eventType": "ticket.updated",
  "ticketId": "3f1c9a12-4e6a-4b0c-9d21-1a2b3c4d5e6f",
  "subject": "Updated subject",
  "description": "Updated description"
}
```

Recommended payload for a ticket note event:

```json
{
  "eventType": "ticket.commented",
  "ticketId": "3f1c9a12-4e6a-4b0c-9d21-1a2b3c4d5e6f",
  "noteId": "7b2e4f10-8c3d-4a51-9e77-2c6b8d1a4f03",
  "noteBody": "Note text"
}
```

Recommended payload for a ticket attachment event:

```json
{
  "eventType": "ticket.attachment.created",
  "ticketId": "3f1c9a12-4e6a-4b0c-9d21-1a2b3c4d5e6f",
  "attachmentId": "9d4a6c21-5f18-4e73-b0c9-2e7f8a1d6c35",
  "fileName": "example.pdf"
}
```

The incoming flow also accepts common Matrix42-style key variants such as `Id`, `TicketId`, `entity.Id`, `note.Text`, `comment.Text`, `attachment.Id` and `attachment.FileName`, plus nested `payload` objects, but the payloads above are the intended contract. An `eventType` of `insert` or `update` is normalized to `ticket.created` and `ticket.updated`.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. Incoming Matrix42 events also check Jira issue properties before creating comments or attachments.

On the Matrix42 side, scope the workflow trigger so it does not fire for changes made by the integration account (for example by excluding the account used for the `basic-auth-matrix42` credential from the workflow's change-author condition). Without that condition, updates iHub writes into Matrix42 are sent straight back to Jira.

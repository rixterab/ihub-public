# Jira <-> Freshdesk Ticket Sync

This template installs two iHub flows that keep Jira issues and Freshdesk tickets linked in both directions.

- `Freshdesk Outgoing Sync`: Jira -> Freshdesk, triggered by Jira webhooks.
- `Freshdesk Incoming Sync`: Freshdesk -> Jira, triggered by Freshdesk automation webhooks calling an iHub custom webhook URL.

The sync stores the Freshdesk ticket ID in a Jira custom field. That field is the link key used by both flows to find the matching record. iHub is the master orchestration point: Freshdesk sends webhook data to iHub, and iHub decides whether to create, update, comment on, or attach files to Jira issues.

## What The Template Syncs

From Jira to Freshdesk:

- Creates a Freshdesk ticket when a Jira issue is created and the Jira issue does not already have a Freshdesk ticket ID.
- Stores the created Freshdesk ticket ID back on the Jira issue.
- Updates the Freshdesk ticket subject and description when the Jira issue is updated.
- Adds Jira comments to the linked Freshdesk ticket as notes.
- Uploads Jira attachments to the linked Freshdesk ticket as note attachments.
- Stores synced Freshdesk comment and attachment IDs as Jira issue properties using keys like `ihub-<freshdesk-id>`.

From Freshdesk to Jira:

- Searches Jira for an issue whose Freshdesk ticket ID custom field matches the incoming Freshdesk ticket ID.
- Creates a Jira issue when Freshdesk sends a `ticket.created` event and no matching Jira issue exists.
- Updates the linked Jira issue from `ticket.updated` events.
- Adds Freshdesk conversation bodies to the matching Jira issue as comments.
- Downloads Freshdesk attachments and uploads them to the matching Jira issue.
- Stores incoming Freshdesk comment and attachment IDs as Jira issue properties using keys like `ihub-<freshdesk-id>` to avoid duplicates.

## Freshdesk API Calls

The outgoing flow uses Freshdesk API v2 endpoints:

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Ticket | `POST` | `{{_flow.FRESHDESK_URL}}/api/v2/tickets` |
| Update Ticket | `PUT` | `{{_flow.FRESHDESK_URL}}/api/v2/tickets/{{issue.fields.customfield_10153}}` |
| Add Note | `POST` | `{{_flow.FRESHDESK_URL}}/api/v2/tickets/{{issue.fields.customfield_10153}}/notes` |
| Add Note With Attachment | `POST` | `{{_flow.FRESHDESK_URL}}/api/v2/tickets/{{issue.fields.customfield_10153}}/notes` |

Freshdesk authenticates with basic auth where the username is the Freshdesk API key and the password is typically `X`.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Freshdesk ticket ID. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://your-domain.freshdesk.com` | Outgoing flow variable `FRESHDESK_URL` | Replace with the customer's Freshdesk account URL. |
| `jira-sync@example.com` | Outgoing flow variable `FRESHDESK_REQUESTER_EMAIL` | Replace with the requester email used when Jira creates Freshdesk tickets. |
| `2` | Outgoing flow variable `FRESHDESK_DEFAULT_PRIORITY` | Replace if Jira-created Freshdesk tickets should use another default priority. |
| `2` | Outgoing flow variable `FRESHDESK_DEFAULT_STATUS` | Replace if Jira-created Freshdesk tickets should use another default status. |
| `2` | Outgoing flow variable `FRESHDESK_TICKET_SOURCE` | Replace if Jira-created Freshdesk tickets should use another Freshdesk source value. |
| `true` | Outgoing flow variable `FRESHDESK_COMMENT_PRIVATE` | Set to `false` if Jira comments should become public Freshdesk notes. |
| `FD` | Incoming flow variable `SPACE` | Replace with the Jira project key where Freshdesk-created tickets should create/find issues. |
| `Task` | Incoming flow variable `ISSUE_TYPE` | Replace if the target Jira project does not use `Task`. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular comment and attachment sync. |

## Places To Update The Jira Custom Field

Update all references to the Freshdesk ticket ID field, not only the create action.

In `freshdesk_outgoing_sync.json`:

- `Create Ticket in Freshdesk` condition: checks `$.issue.fields.customfield_10153` is empty before creating a ticket.
- `Store Freshdesk Ticket ID on Jira Issue` body: writes the created Freshdesk ticket ID back to `customfield_10153`.
- `Add Jira Comment to Freshdesk Ticket` URL, flow variable, and condition: uses `issue.fields.customfield_10153`.
- `Update Freshdesk Ticket from Jira Issue` URL, flow variable, and condition: uses `issue.fields.customfield_10153`.
- `Upload Jira Attachment to Freshdesk Ticket` URL and condition: uses `issue.fields.customfield_10153`.

In `freshdesk_incoming_sync.json`:

- `Find Jira Issue by Freshdesk Ticket ID` JQL: searches `cf[10153]` for the incoming Freshdesk ticket ID.
- `Create Jira Issue from Freshdesk Ticket` body: writes the incoming Freshdesk ticket ID into `customfield_10153`.

## Freshdesk Webhook Setup

After the template is instantiated in iHub, copy the trigger URL from the `Freshdesk Incoming Sync` custom webhook trigger.

In Freshdesk:

1. Go to `Admin` -> `Workflows` -> `Automations`.
2. Add a Ticket Creation rule that triggers a webhook to the iHub URL.
3. Add a Ticket Updates rule that triggers a webhook to the same iHub URL.
4. Use `POST`, JSON encoding, and an advanced/custom body.
5. Add Freshdesk authentication only if the iHub custom webhook is configured to require it.

Recommended payload for a ticket creation event:

```json
{
  "eventType": "ticket.created",
  "ticket": {
    "id": "{{ticket.id}}",
    "subject": "{{ticket.subject}}",
    "description": "{{ticket.description}}",
    "status": "{{ticket.status}}",
    "priority": "{{ticket.priority}}"
  }
}
```

Recommended payload for a ticket update event:

```json
{
  "eventType": "ticket.updated",
  "ticket": {
    "id": "{{ticket.id}}",
    "subject": "{{ticket.subject}}",
    "description": "{{ticket.description}}",
    "status": "{{ticket.status}}",
    "priority": "{{ticket.priority}}",
    "modified_on": "{{ticket.modified_on}}"
  }
}
```

Recommended payload for a Freshdesk conversation/comment event:

```json
{
  "eventType": "ticket.commented",
  "ticket": {
    "id": "{{ticket.id}}"
  },
  "conversation": {
    "id": "{{ticket.id}}-{{ticket.modified_on}}",
    "body": "{{ticket.latest_public_comment}}"
  }
}
```

Recommended payload for an attachment event:

```json
{
  "eventType": "ticket.attachment.created",
  "ticket": {
    "id": "{{ticket.id}}"
  },
  "attachment": {
    "id": "freshdesk-attachment-id",
    "name": "example.pdf",
    "attachment_url": "https://..."
  }
}
```

The incoming flow accepts common variants such as top-level `ticketId`, `freshdeskTicketId`, `commentBody`, `commentId`, `attachmentUrl`, `attachmentId`, nested `ticket`, `conversation`, `note`, and `attachment` objects, and first entries in `attachments` arrays. If Freshdesk cannot provide a stable conversation ID, the flow derives one from ticket ID, modified timestamp, and comment body.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. Incoming Freshdesk events check Jira issue properties before creating comments or attachments. This prevents updates created by one side of the sync from being immediately sent back to the source side.

# Jira <-> Zendesk Ticket Sync

This template installs two iHub flows that keep Jira issues and Zendesk tickets linked in both directions. iHub is the master orchestration point for webhook handling: Zendesk sends ticket events to an iHub custom webhook URL, and iHub decides whether to create, update, comment on, or attach files to Jira issues.

- `Zendesk Outgoing Sync`: Jira -> Zendesk, triggered by Jira webhooks.
- `Zendesk Incoming Sync`: Zendesk -> Jira, triggered by Zendesk webhooks calling an iHub custom webhook URL.

The sync stores the Zendesk ticket ID in a Jira custom field. That field is the link key used by both flows to find the matching record.

## What The Template Syncs

From Jira to Zendesk:

- Creates a Zendesk ticket when a Jira issue is created and the Jira issue does not already have a Zendesk ticket ID.
- Stores the created Zendesk ticket ID back on the Jira issue.
- Sets the Zendesk ticket `external_id` to the Jira issue key.
- Updates the linked Zendesk ticket subject when the Jira issue is updated.
- Adds Jira comments to the linked Zendesk ticket as ticket comments.
- Uploads Jira attachments to Zendesk, then attaches them to a new Zendesk ticket comment.
- Stores synced Zendesk comment and attachment IDs as Jira issue properties using keys like `ihub-<zendesk-id>`.

From Zendesk to Jira:

- Searches Jira for an issue whose Zendesk ticket ID custom field matches the incoming Zendesk ticket ID.
- Creates a Jira issue when Zendesk sends a `ticket.created` event and no matching Jira issue exists.
- Updates the linked Jira issue from ticket update events.
- Adds Zendesk ticket comments to the matching Jira issue as comments.
- Downloads Zendesk attachments and uploads them to the matching Jira issue.
- Stores incoming Zendesk comment and attachment IDs as Jira issue properties using keys like `ihub-<zendesk-id>` to avoid duplicates.

## Zendesk API Calls

The outgoing flow uses Zendesk Support API v2 endpoints:

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Ticket | `POST` | `{{_flow.ZENDESK_URL}}/api/v2/tickets` |
| Update Ticket Subject | `PUT` | `{{_flow.ZENDESK_URL}}/api/v2/tickets/{{issue.fields.customfield_10153}}` |
| Add Ticket Comment | `PUT` | `{{_flow.ZENDESK_URL}}/api/v2/tickets/{{issue.fields.customfield_10153}}` |
| Upload Attachment | `POST` | `{{_flow.ZENDESK_URL}}/api/v2/uploads?filename={{filename}}` |
| Add Comment With Uploaded Attachment | `PUT` | `{{_flow.ZENDESK_URL}}/api/v2/tickets/{{issue.fields.customfield_10153}}` |

Zendesk ticket comments are created by adding a `ticket.comment` object while creating or updating a ticket. Attachments are uploaded first to get an upload token, then the token is added to a new ticket comment.

The template uses a Basic Auth credential for Zendesk. For Zendesk API token auth, set the username to `{email_address}/token` and the password to the Zendesk API token. OAuth is preferred for distributed/multi-customer integrations, but Basic Auth with an API token is still useful for customer-owned internal flows.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Zendesk ticket ID. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://your-subdomain.zendesk.com` | Outgoing flow variable `ZENDESK_URL` | Replace with the customer's Zendesk account URL. |
| `Jira Sync` | Outgoing flow variable `ZENDESK_REQUESTER_NAME` | Replace with the requester name used when Jira creates Zendesk tickets. |
| `jira-sync@example.com` | Outgoing flow variable `ZENDESK_REQUESTER_EMAIL` | Replace with the requester email used when Jira creates Zendesk tickets. |
| `normal` | Outgoing flow variable `ZENDESK_DEFAULT_PRIORITY` | Replace if Jira-created Zendesk tickets should use another default priority. |
| `new` | Outgoing flow variable `ZENDESK_DEFAULT_STATUS` | Replace if Jira-created Zendesk tickets should use another default status. |
| `true` | Outgoing flow variable `ZENDESK_INITIAL_COMMENT_PUBLIC` | Set to `false` if Jira descriptions should create internal Zendesk notes. |
| `false` | Outgoing flow variable `ZENDESK_COMMENT_PUBLIC` | Set to `true` if Jira comments and attachment comments should be public Zendesk comments. |
| `ZD` | Incoming flow variable `SPACE` | Replace with the Jira project key where Zendesk-created tickets should create/find issues. |
| `Task` | Incoming flow variable `ISSUE_TYPE` | Replace if the target Jira project does not use `Task`. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular comment and attachment sync. |

## Places To Update The Jira Custom Field

Update all references to the Zendesk ticket ID field, not only the create action.

In `zendesk_outgoing_sync.json`:

- `Create Ticket in Zendesk` condition: checks `$.issue.fields.customfield_10153` is empty before creating a ticket.
- `Store Zendesk Ticket ID on Jira Issue` body: writes the created Zendesk ticket ID back to `customfield_10153`.
- `Add Jira Comment to Zendesk Ticket` URL, flow variable, and condition: uses `issue.fields.customfield_10153`.
- `Update Zendesk Ticket from Jira Issue` URL, flow variable, and condition: uses `issue.fields.customfield_10153`.
- `Upload Jira Attachment to Zendesk` and `Add Jira Attachment Comment to Zendesk Ticket`: use `issue.fields.customfield_10153`.

In `zendesk_incoming_sync.json`:

- `Find Jira Issue by Zendesk Ticket ID` JQL: searches `cf[10153]` for the incoming Zendesk ticket ID.
- `Create Jira Issue from Zendesk Ticket` body: writes the incoming Zendesk ticket ID into `customfield_10153`.

## Zendesk Webhook Setup

After the template is instantiated in iHub, copy the trigger URL from the `Zendesk Incoming Sync` custom webhook trigger.

In Zendesk:

1. Go to `Admin Center` -> `Apps and integrations` -> `Webhooks`.
2. Create an active webhook pointing to the iHub URL. Use `POST` and JSON request format.
3. Go to `Objects and rules` -> `Business rules` -> `Triggers`.
4. Create or update ticket triggers that notify the active webhook for ticket creation, ticket updates, comments, and attachments.
5. Add Zendesk webhook authentication only if the iHub custom webhook is configured to require it.

Recommended payload for a ticket creation event:

```json
{
  "eventType": "ticket.created",
  "ticket": {
    "id": "{{ticket.id}}",
    "subject": "{{ticket.title}}",
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
    "subject": "{{ticket.title}}",
    "description": "{{ticket.description}}",
    "status": "{{ticket.status}}",
    "priority": "{{ticket.priority}}",
    "updated_at": "{{ticket.updated_at_with_timestamp}}"
  }
}
```

Recommended payload for a Zendesk comment event:

```json
{
  "eventType": "ticket.commented",
  "ticket": {
    "id": "{{ticket.id}}"
  },
  "comment": {
    "id": "{{ticket.id}}-{{ticket.updated_at_with_timestamp}}",
    "body": "{{ticket.latest_comment_html}}"
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
    "id": "zendesk-attachment-id",
    "filename": "example.pdf",
    "content_url": "https://..."
  }
}
```

The incoming flow also accepts Zendesk event webhook payloads with `type` values such as `zen:event-type:ticket.created`, `zen:event-type:ticket.comment_added`, and `zen:event-type:ticket.attachment_linked_to_comment`. It reads ticket data from `detail`, comment data from `event.comment`, and attachment data from `event.comment.attachment`. Native Zendesk event payloads include stable comment and attachment IDs; trigger-based payloads can use a ticket/timestamp composite ID, as shown above.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. Incoming Zendesk events check Jira issue properties before creating comments or attachments. This prevents updates created by one side of the sync from being immediately sent back to the source side.

On the Zendesk side, restrict triggers so they do not call the iHub webhook for changes made by the Zendesk integration user where possible.

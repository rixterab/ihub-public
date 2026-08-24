# Jira <-> ServiceNow Ticket Sync

This template installs two iHub flows that keep Jira issues and ServiceNow tickets linked in both directions. iHub is the master of the integration: it listens to ServiceNow webhooks and to Jira webhooks, and drives every create and update on both sides.

- `ServiceNow Incoming Sync`: ServiceNow -> Jira, triggered by a ServiceNow Business Rule or Flow Designer REST step posting to an iHub custom webhook URL.
- `ServiceNow Outgoing Sync`: Jira -> ServiceNow, triggered by Jira webhooks.

The sync stores the ServiceNow record `sys_id` in a Jira custom field. That field is the link key used by both flows to find the matching record.

## What The Template Syncs

From ServiceNow to Jira:

- Searches Jira for an issue whose ServiceNow sys_id custom field matches the incoming `sys_id`.
- Creates a Jira issue when ServiceNow sends a `ticket.created` event and no matching Jira issue exists.
- Updates the matching Jira issue summary and description on a `ticket.updated` event.
- Adds ServiceNow journal comments to the matching Jira issue.
- Downloads ServiceNow attachments by attachment `sys_id` and uploads them to the matching Jira issue.
- Stores incoming ServiceNow comment and attachment sys_ids as Jira issue properties using keys like `ihub-<sys_id>`.

From Jira to ServiceNow:

- Creates a ServiceNow ticket when a Jira issue is created and the Jira issue does not already have a ServiceNow sys_id.
- Stores the created ServiceNow `sys_id` back on the Jira issue and sets `correlation_id` to the Jira issue key.
- Updates the ServiceNow `short_description` and `description` when the Jira issue is updated.
- Adds Jira comments to the linked ServiceNow ticket through the journal `comments` field.
- Uploads Jira attachments to the linked ServiceNow ticket using the Attachment API.
- Stores synced comment and attachment ids as Jira issue properties using keys like `ihub-<id>` to avoid duplicate processing.

## ServiceNow API Calls

Both flows use OAuth2 and the current ServiceNow REST APIs: Table API `v2` and Attachment API `/api/now/attachment`.

| Action | Method | Endpoint |
| --- | --- | --- |
| Create ticket | `POST` | `{{_flow.SN_URL}}/api/now/{{_flow.SN_API_VERSION}}/table/{{_flow.SN_TABLE}}` |
| Update ticket | `PATCH` | `{{_flow.SN_URL}}/api/now/{{_flow.SN_API_VERSION}}/table/{{_flow.SN_TABLE}}/{{issue.fields.customfield_10153}}` |
| Comment ticket | `PATCH` | `{{_flow.SN_URL}}/api/now/{{_flow.SN_API_VERSION}}/table/{{_flow.SN_TABLE}}/{{issue.fields.customfield_10153}}` |
| Upload attachment | `POST` | `{{_flow.SN_URL}}/api/now/attachment/upload` |
| Download attachment | `GET` | `{{_flow.SN_URL}}/api/now/attachment/{{serviceNowAttachmentId}}/file` |

Comments are written by patching the ticket's journal input field `comments` (Additional comments). To write to `work_notes` instead, change the field name in the `Comment ServiceNow Ticket` action body.

Attachment upload uses the multipart endpoint `/api/now/attachment/upload` with the fields `table_name`, `table_sys_id` and `uploadFile`, so ServiceNow links the uploaded file to the ticket record.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the ServiceNow `sys_id`. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://your-instance.service-now.com` | Incoming and outgoing `SN_URL` flow variable | Replace with the customer's ServiceNow instance URL. |
| `v2` | Incoming and outgoing `SN_API_VERSION` flow variable | Replace if the instance must use another Table API version. |
| `incident` | Incoming and outgoing `SN_TABLE` flow variable | Replace with the table holding the tickets, for example `sn_customerservice_case` or `sc_req_item`. |
| `integration` | Outgoing `SN_CONTACT_TYPE` flow variable | Must match an allowed `contact_type` choice in the instance. |
| `3` | Outgoing `SN_IMPACT` and `SN_URGENCY` flow variables | Adjust the default impact and urgency of tickets created from Jira. |
| `SN` | Incoming `SPACE` flow variable | Replace with the Jira project key where ServiceNow tickets should create/find issues. |
| `Task` | Incoming `ISSUE_TYPE` flow variable | Replace if the target Jira project does not use `Task`. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular comment and attachment sync. |
| `your-instance` in the OAuth2 URLs | `manifest.json` credential definition | Replace with the customer's instance name before importing, or edit the credential after import. |

## ServiceNow Webhook Payload

After the template is instantiated in iHub, copy the trigger URL from the `ServiceNow Incoming Sync` custom webhook trigger. Configure a ServiceNow Business Rule (or a Flow Designer REST step) to POST JSON payloads to that URL.

Recommended payload for a ticket create event:

```json
{
  "eventType": "ticket.created",
  "table": "incident",
  "sysId": "9c573169c611228700193229fff72400",
  "number": "INC0010001",
  "shortDescription": "Ticket short description",
  "description": "Ticket description"
}
```

Recommended payload for a ticket update event:

```json
{
  "eventType": "ticket.updated",
  "table": "incident",
  "sysId": "9c573169c611228700193229fff72400",
  "number": "INC0010001",
  "shortDescription": "Updated short description",
  "description": "Updated description"
}
```

Recommended payload for a ticket comment event:

```json
{
  "eventType": "ticket.commented",
  "table": "incident",
  "sysId": "9c573169c611228700193229fff72400",
  "commentId": "3f7c9a12c611228700193229fff7ab01",
  "commentBody": "Comment text"
}
```

Recommended payload for a ticket attachment event:

```json
{
  "eventType": "ticket.attachment.created",
  "table": "incident",
  "sysId": "9c573169c611228700193229fff72400",
  "attachmentId": "b1d4f9e2c611228700193229fff7cd02",
  "fileName": "example.pdf"
}
```

The incoming flow also accepts common ServiceNow-style key variants such as `sys_id`, `short_description`, `record.sys_id`, `current.sys_id`, `comments`, `work_notes`, `attachment.sys_id` and `file_name`, plus nested `payload` objects, but the payloads above are the intended contract. An `eventType` of `insert` or `update` is normalized to `ticket.created` and `ticket.updated`.

`commentId` should be the `sys_id` of the `sys_journal_field` entry. `attachmentId` should be the `sys_id` of the `sys_attachment` record.

## OAuth2

The manifest defines one ServiceNow OAuth2 credential: `oauth2-servicenow`. It uses the instance endpoints `/oauth_auth.do` and `/oauth_token.do` with the `useraccount` scope, and is used by every ServiceNow REST call in both flows.

Before this works, register an OAuth application in ServiceNow under `System OAuth > Application Registry` as an `OAuth API endpoint for external clients`, and add the iHub OAuth callback URL as the redirect URL. The integration user tied to the credential needs read and write access to the configured ticket table, to `sys_journal_field` and to `sys_attachment`.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. Incoming ServiceNow events also check Jira issue properties before creating comments or attachments.

On the ServiceNow side, restrict the Business Rule so it does not fire for changes made by the integration user, for example with the condition `current.sys_updated_by != 'ihub.integration'`. Without that condition, updates iHub writes into ServiceNow are sent straight back to Jira.

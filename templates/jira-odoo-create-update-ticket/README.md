# Jira ↔ Odoo Helpdesk Sync

This template installs two iHub flows that synchronize Jira work items and Odoo Helpdesk tickets in both directions, including comments and binary file attachments.

- `Jira to Odoo Helpdesk Sync`: Jira → Odoo, triggered by Jira webhooks.
- `Odoo Helpdesk to Jira Sync`: Odoo → Jira, triggered by an Odoo integration webhook calling an iHub custom webhook URL.

This is a **two-way sync**. It stores the Odoo ticket ID in a Jira single-line text custom field. That field is the durable link used by both flows to find the matching record and prevent duplicate creates.

The Odoo routes used here are an explicit integration contract, not standard Odoo Helpdesk endpoints. The Odoo instance needs a small custom module that exposes the routes and webhook payloads documented below. The original template already targeted `/api/ticket/sync`; this version keeps that route and defines the rest of the contract needed for a complete two-way sync.

## What The Template Does

From Jira to Odoo:

- Creates an Odoo Helpdesk ticket when a Jira work item is created in the configured project and has no Odoo ticket ID.
- Writes the returned Odoo ticket ID to the Jira custom field.
- Updates the linked Odoo ticket name and description when the Jira work item changes.
- Adds Jira comments to the linked Odoo ticket.
- Uploads Jira attachment bytes to the linked Odoo ticket.
- Checks and writes Jira issue properties for comment and attachment IDs so webhook redelivery does not create duplicates.

From Odoo to Jira:

- Normalizes supported Odoo webhook payload variants with scripted variables.
- Searches Jira for a work item whose Odoo ticket ID field matches the incoming ticket ID.
- Creates a Jira work item for a `ticket.created` event when no match exists.
- Updates summary and description for a `ticket.updated` event when a match exists.
- Adds Odoo comments to Jira after checking the Odoo message ID issue property.
- Downloads Odoo attachment bytes and uploads them to Jira after checking the Odoo attachment ID issue property.
- Writes issue properties after successful comment and attachment processing.

## API Calls

### Odoo integration API

| Action                         | Method           | Endpoint                                                      |
| ------------------------------ | ---------------- | ------------------------------------------------------------- |
| Create Odoo Helpdesk Ticket    | `POST`           | `{{ODOO_URL}}/api/ticket/sync`                                |
| Update Odoo Helpdesk Ticket    | `POST`           | `{{ODOO_URL}}/api/ticket/sync`                                |
| Add Jira Comment to Odoo       | `POST`           | `{{ODOO_URL}}/api/ticket/comment`                             |
| Upload Jira Attachment to Odoo | `POST` multipart | `{{ODOO_URL}}/api/ticket/attachment`                          |
| Download Odoo Attachment       | `GET`            | `{{ODOO_URL}}/api/ticket/attachment/{attachment_id}/download` |

All Odoo routes must accept `Authorization: Bearer <token>`. The create route must return:

```json
{ "ticket_id": "123" }
```

The comment route must return:

```json
{ "comment_id": "456" }
```

The attachment upload route must return:

```json
{ "attachment_id": "789" }
```

The integration module should enforce these idempotency keys as unique per ticket:

- `jira_key` for ticket creation.
- `jira_comment_id` for comments.
- `jira_attachment_id` for attachments.

That server-side uniqueness complements the Jira link-field and issue-property guards. It also protects against a retry that reaches Odoo but fails before iHub can write the corresponding Jira property.

### Jira Cloud REST API

| Action                                 | Method           | Endpoint                                                |
| -------------------------------------- | ---------------- | ------------------------------------------------------- |
| Find Jira Work Item by Odoo Ticket ID  | `POST`           | `/rest/api/3/search/jql`                                |
| Create Jira Work Item from Odoo Ticket | `POST`           | `/rest/api/3/issue`                                     |
| Update Jira Work Item from Odoo Ticket | `PUT`            | `/rest/api/3/issue/{issueKey}`                          |
| Check comment/attachment sync property | `GET`            | `/rest/api/3/issue/{issueKey}/properties/{propertyKey}` |
| Add Odoo Comment to Jira               | `POST`           | `/rest/api/3/issue/{issueKey}/comment`                  |
| Upload Odoo Attachment to Jira         | `POST` multipart | `/rest/api/3/issue/{issueKey}/attachments`              |
| Store comment/attachment sync property | `PUT`            | `/rest/api/3/issue/{issueKey}/properties/{propertyKey}` |

Jira attachment uploads include `X-Atlassian-Token: no-check`, as required for Jira's multipart attachment endpoint.

## Body Format And Fidelity

Jira Cloud descriptions and comments are ADF objects. Odoo Helpdesk descriptions and chatter messages are HTML in this integration contract.

- Jira → Odoo uses `adfToHTML` for descriptions and comments. Common paragraphs, lists, emphasis, and links are preserved as HTML. Jira-only ADF nodes may be simplified.
- Odoo → Jira uses `htmlToADF` for descriptions and comments. HTML that has no Jira ADF equivalent may be simplified or dropped.
- Ticket names and summaries are sent as plain text.

Free-text values use raw Handlebars interpolation to avoid HTML entities such as `&#x27;` appearing in Odoo or Jira. As elsewhere in this repository, a double quote inside a summary or another free-text value can still break the surrounding JSON request body. If that is common in the customer data, the Odoo module should accept a safer encoded field or iHub should gain a supported JSON-string escaping helper before this template is enabled.

Attachments are copied as binary files in both directions:

- Jira → Odoo uses the Jira attachment iterator and `attachments.unsentFile`, then sends multipart form data to the Odoo module.
- Odoo → Jira downloads the file from the authenticated Odoo download route into iHub `INPUT_FILES`, then uploads `input.files` to Jira as multipart form data.

The Odoo module must return the original bytes and a safe filename. Odoo and Jira upload-size, file-type, storage, malware-scanning, and retention rules still apply. An attachment rejected by either system is not converted into a link or placeholder.

## Credential

The manifest defines `token-odoo`, a `TOKEN` credential. Create a dedicated Odoo integration user and a long random token in the custom integration module. When configuring the credential in iHub, paste only the raw token:

```text
your-long-random-token
```

Do **not** paste `Bearer `. A `TOKEN` credential adds the `Authorization: Bearer` prefix automatically.

The integration user needs only the permissions required to create and update Helpdesk tickets, post internal messages, create and read ticket attachments, and read the configured Helpdesk team. Its numeric Odoo user ID is also used by the incoming actor filter, so do not use a normal employee account whose ownership may change.

## Odoo Integration Module And Webhook Setup

Implement the API contract before enabling either flow.

1. Expose the five authenticated routes listed under API Calls.
2. On `/api/ticket/sync`, treat `action: create` as an idempotent upsert keyed by `jira_key`, and return the existing ticket ID if the same Jira key is delivered again.
3. On `/api/ticket/comment`, deduplicate by `jira_comment_id` within the Odoo ticket.
4. On `/api/ticket/attachment`, accept multipart fields `file`, `odoo_id`, `jira_key`, `jira_attachment_id`, and `file_name`; deduplicate by `jira_attachment_id` within the ticket.
5. On the download route, verify that the authenticated integration user may read the attachment and that the attachment belongs to a Helpdesk ticket.
6. Instantiate the template in iHub and copy the custom webhook URL from `Odoo Helpdesk to Jira Sync`.
7. Configure the Odoo module to `POST` ticket, comment, and attachment events to that URL.
8. Emit webhook events only after the Odoo transaction commits. The iHub trigger returns a fixed JSON `{}` response; retry non-2xx deliveries with backoff while preserving the same ticket, comment, and attachment IDs.
9. Confirm all payload paths with real deliveries in the iHub flow log before enabling both directions.

The preferred webhook shapes are:

```json
{
  "event_type": "ticket.created",
  "ticket_id": "123",
  "team_id": "1",
  "actor_id": "42",
  "ticket": {
    "name": "Printer unavailable",
    "description_html": "<p>The third-floor printer is offline.</p>"
  }
}
```

```json
{
  "event_type": "ticket.updated",
  "ticket_id": "123",
  "team_id": "1",
  "actor_id": "42",
  "ticket": {
    "name": "Printer unavailable",
    "description_html": "<p>The third-floor printer is offline.</p>"
  }
}
```

```json
{
  "event_type": "comment.created",
  "ticket_id": "123",
  "team_id": "1",
  "actor_id": "42",
  "comment": {
    "id": "456",
    "body_html": "<p>A technician is on the way.</p>"
  }
}
```

```json
{
  "event_type": "attachment.created",
  "ticket_id": "123",
  "team_id": "1",
  "actor_id": "42",
  "attachment": {
    "id": "789",
    "file_name": "printer-error.jpg"
  }
}
```

The incoming scripted variables also tolerate the payload inside a top-level `payload` object and several common flat/nested key variants. The canonical shapes above are the tested contract; tolerance is not a reason to omit fields.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Odoo ticket ID. The template uses `customfield_10153` in Jira bodies and events and `cf[10153]` in JQL. Replace both forms with the customer's actual field ID before enabling the flows.

| Value in template                            | Where                                        | Customer action                                                                     |
| -------------------------------------------- | -------------------------------------------- | ----------------------------------------------------------------------------------- |
| `ODOO`                                       | Both flows, `SPACE`                          | Replace with the Jira project key to synchronize. Use the same value in both flows. |
| `https://your-instance.odoo.com`             | Both flows, `ODOO_URL`                       | Replace with the Odoo base URL, without a trailing slash.                           |
| `1`                                          | Both flows, `ODOO_HELPDESK_TEAM_ID`          | Replace with the numeric Odoo Helpdesk team ID. Use the same value in both flows.   |
| `Task`                                       | Incoming flow, `ISSUE_TYPE`                  | Replace with the Jira work item type created for Odoo tickets.                      |
| `712020:replace-with-integration-account-id` | Outgoing flow, `JIRA_INTEGRATION_ACCOUNT_ID` | Replace with the Atlassian account ID used by iHub for Jira writes.                 |
| `1`                                          | Incoming flow, `ODOO_INTEGRATION_USER_ID`    | Replace with the numeric Odoo user ID used by the integration API.                  |
| `customfield_10153`                          | Both flow files                              | Replace with the Jira field key that stores the Odoo ticket ID.                     |
| `cf[10153]`                                  | Incoming JQL search                          | Replace with the numeric JQL form of the same Jira field.                           |
| `token-odoo`                                 | Manifest credential                          | Configure the raw Odoo integration token in iHub; do not include `Bearer `.         |

### Places To Update The Jira Custom Field

In `jira_odoo_helpdesk_outgoing.json`:

- `Create Odoo Helpdesk Ticket` checks whether `issue.fields.customfield_10153` is empty.
- `Store Odoo Ticket ID on Jira` writes `customfield_10153`.
- `Update Odoo Helpdesk Ticket` reads and checks `issue.fields.customfield_10153`.
- `Check Jira Comment Sync Property` reads and checks `issue.fields.customfield_10153`.
- `Check Jira Attachment Sync Property` reads and checks `issue.fields.customfield_10153`.
- `Add Jira Comment to Odoo` and `Upload Jira Attachment to Odoo` send `issue.fields.customfield_10153` as `odoo_id`.

In `odoo_helpdesk_jira_incoming.json`:

- `Find Jira Work Item by Odoo Ticket ID` searches `cf[10153]`.
- `Create Jira Work Item from Odoo Ticket` writes `customfield_10153`.

## Loop Prevention And Idempotency

The sync uses five layers of protection:

1. The Jira create path only runs when the Odoo ticket ID custom field is empty. The Odoo incoming create path searches Jira before creating.
2. Jira-originated updates, comments, and attachments are blocked when the acting Jira account is `JIRA_INTEGRATION_ACCOUNT_ID`.
3. Odoo-originated events are blocked when `actor_id` is `ODOO_INTEGRATION_USER_ID`.
4. Comment and attachment IDs are checked and stored as Jira issue properties before and after copying content.
5. The Odoo module must enforce uniqueness for `jira_key`, `jira_comment_id`, and `jira_attachment_id`.

Writing the Odoo ticket ID back to Jira is itself a Jira update. The Jira actor filter drops that update because iHub made it. Likewise, Odoo must attribute API writes to the configured integration user so its outgoing webhook is dropped by the incoming flow.

Actor locations differ between Jira event types. This template follows the established repository paths: `payload.user.accountId` for updates, `comment.author.accountId` for comments, and `payload.author.accountId` for attachments. Verify those paths using real events from the customer's Jira site before relying on the loop guard.

## Webhook Security

The incoming flow uses `CUSTOM_WEBHOOK_TRIGGER_TYPE`.

**iHub cannot verify Odoo webhook signatures.** The flow therefore cannot cryptographically prove that a request came from Odoo. Treat the iHub webhook URL as a credential: do not commit it, paste it into tickets or chat, include it in screenshots, or share it beyond the integration administrators. If it leaks, recreate the trigger to rotate the URL and update the Odoo module.

Anyone who learns the URL can submit a fabricated payload and cause the Jira integration user to create or modify work items, add comments, or fetch an attachment ID from the configured Odoo instance. The flow checks `team_id` against `ODOO_HELPDESK_TEAM_ID` as a weak authenticity and scope filter. It also rejects the configured integration actor to prevent loops. Neither check substitutes for signature verification, because both values are present in the request body and can be forged.

The attachment download URL is constructed from `ODOO_URL` and `attachment_id`; the webhook cannot supply an arbitrary download host. The Odoo download route must still authorize access and must not return attachments from unrelated records.

## Conflict Policy

Title/name and description use last-write-wins semantics. There is no field-level merge, version check, timestamp comparison, or conflict marker. If both records change at nearly the same time, whichever event iHub processes last overwrites the other side.

## Limitations

- The template requires a custom Odoo module implementing the documented routes and webhook emitter. Installing the JSON flows alone is not sufficient.
- Status/stage is not synchronized. Jira status changes require Jira transitions, while Odoo Helpdesk stages are database-specific records. Add an explicit mapping table and transition actions before enabling status sync.
- Assignees, customers, followers, priorities, tags, SLAs, activities, time entries, subtasks, and linked records are not synchronized.
- Only one Jira project and one Odoo Helpdesk team are handled per flow pair. Install/configure another pair for a second mapping.
- Only newly delivered comments and attachments are copied. The template does not backfill historical chatter or files when an existing ticket is first linked.
- Deleting or editing a comment or attachment does not delete or edit the copy on the other side.
- Odoo's integration endpoint must return the exact response keys documented above. An HTTP success without the expected ID stops the write-back/property branch and should be treated as an application error in the Odoo module logs.
- Jira/Odoo file-size limits and request timeouts can prevent large attachment transfers. There is no chunked upload or resumable transfer.
- The Jira webhook actor paths and the Odoo payload variants must be verified against real deliveries before production use. The canonical Odoo payloads in this README are the contract the custom module should emit.

In Odoo 18 you can do this with an Automation Rule on the Helpdesk Ticket model and use Send Webhook Notification as the action. Odoo sends an HTTP POST to your iHub webhook URL with the fields you select.

For your case I would configure it like this:

1. Open Helpdesk → Studio → Automations
2. Create a new automation.
3. Model: Helpdesk Ticket (helpdesk.ticket)
4. Trigger: use On create and edit / On save depending on what your Odoo UI shows.
5. Action: Send Webhook Notification
6. URL: paste the iHub incoming webhook URL.
7. Select the fields you want Odoo to send.

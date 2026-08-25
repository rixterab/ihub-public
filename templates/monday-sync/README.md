# Jira <-> monday.com Item Sync

This template installs two iHub flows that keep Jira issues and monday.com items linked in both directions. iHub is the master of the integration: it listens to monday.com webhooks and to Jira webhooks, and drives every create and update on both sides.

- `Monday Incoming Sync`: monday.com -> Jira, triggered by a monday.com board webhook posting to an iHub custom webhook URL.
- `Monday Outgoing Sync`: Jira -> monday.com, triggered by Jira webhooks.

monday.com has no ticket concept, so the object model is **one Jira issue <-> one monday item** on a single configured board. The sync stores the monday item ID in a Jira custom field, and the Jira issue key in a monday text column. Those two values are the link key used by both flows to find the matching record.

This is the first GraphQL template in this repository. Read [GraphQL Instead Of REST](#graphql-instead-of-rest) before editing either flow.

## What The Template Syncs

From monday.com to Jira:

- Searches Jira for an issue whose monday item ID custom field matches the incoming item (pulse).
- Creates a Jira issue on a `create_item` event when no matching Jira issue exists, then writes the new Jira key back into the monday text column.
- Updates the matching Jira issue summary on a `change_name` event.
- Updates the matching Jira issue description on a `change_column_value` event for the configured long text column.
- Adds monday updates to the matching Jira issue as comments on a `create_update` event.
- Downloads monday assets through their signed `public_url` and uploads them to the matching Jira issue.
- Stores incoming monday update and asset IDs as Jira issue properties using keys like `ihub-<monday-id>`.

From Jira to monday.com:

- Looks up an existing monday item carrying the Jira issue key before creating anything, so a retried or re-linked issue does not create a duplicate item.
- Creates a monday item in the configured board and group when a Jira issue is created and the Jira issue does not already have a monday item ID.
- Stores the created monday item ID back on the Jira issue.
- Updates the monday item name from the Jira summary when the Jira issue is updated.
- Writes the Jira description into the monday long text column when the Jira issue is updated.
- Adds Jira comments to the linked monday item as updates.
- Uploads Jira attachments to the monday files column on the linked item.
- Stores synced comment and asset IDs as Jira issue properties using keys like `ihub-<id>` to avoid duplicate processing.

## GraphQL Instead Of REST

Every other template in this repository is REST, where each action has its own URL and method. monday.com has no REST API. **Every** monday call is `POST https://api.monday.com/v2` with a body of `{"query": "...", "variables": {...}}`, so the URL and method columns of an action tell a reader nothing. The action `name` and `description` carry all the meaning.

Three consequences shape both flows:

**Responses nest under `data.<operation_name>`.** iHub exposes the HTTP response body at `$.response.data`, and monday wraps its own payload in a second `data` object, so flow variable JSONPaths carry a double `data`, for example `$.response.data.data.create_item.id` and `$.response.data.data.items_page_by_column_values.items[0].id`.

**GraphQL fails with HTTP 200.** A failed monday call returns status 200 with an `errors` array in the body, so status-code conditions never catch it. Actions that must only run after a monday call succeeded carry a `$.response.data.errors` `EQUALS` `""` condition instead.

**Column values are double encoded.** `create_item` and `change_multiple_column_values` take `column_values` as a **JSON string**, not a JSON object, so the value is JSON encoded inside the action body, which is itself a JSON string in the flow file. In the raw flow file that reads:

```
\"columnValues\": \"{\\\"text__1\\\": \\\"{{issue.key}}\\\"}\"
```

which monday receives as `column_values: "{\"text__1\": \"ABC-1\"}"`. All three levels are exercised in `Create Monday Item`, which is the only action that needs them. Everywhere else the flows use GraphQL `variables` plus `change_simple_column_value(board_id:, item_id:, column_id:, value:)`, whose `value` is a plain `String!`. That keeps free text such as the Jira summary and description out of the double encoded string entirely, which is where escaping bugs come from.

## Monday API Calls

Both flows authenticate with a monday API token in the `Authorization` header and send the API version in the `API-Version` header. Every call below is `POST https://api.monday.com/v2` except the file upload, which uses the separate multipart endpoint.

| Action | GraphQL operation | Endpoint |
| --- | --- | --- |
| Find Monday Item by Jira Key | `items_page_by_column_values(board_id:, columns: [{column_id:, column_values:}])` | `https://api.monday.com/v2` |
| Create Monday Item | `create_item(board_id:, group_id:, item_name:, column_values:)` | `https://api.monday.com/v2` |
| Update Monday Item Name | `change_simple_column_value(column_id: "name")` | `https://api.monday.com/v2` |
| Update Monday Item Description Column | `change_simple_column_value(column_id: $columnId)` | `https://api.monday.com/v2` |
| Write Jira Key to Monday Item | `change_simple_column_value(column_id: $columnId)` | `https://api.monday.com/v2` |
| Add Update to Monday Item | `create_update(item_id:, body:)` | `https://api.monday.com/v2` |
| Get Monday Asset Public URL | `assets(ids:) { public_url }` | `https://api.monday.com/v2` |
| Upload Attachment to Monday Item | `add_file_to_column(item_id:, column_id:, file:)` | `https://api.monday.com/v2/file` |

Comments are monday **updates**, created with `create_update(item_id:, body:)`. Update bodies are HTML, so both flows use the `{{{adfToHTML ...}}}` and `{{{htmlToADF ...}}}` helpers on the comment body, the same way the ServiceNow template does.

The Jira description is written into a monday **long text column** rather than the first update. Updates are append only and cannot be edited, so they cannot represent a description that changes on the Jira side; a long text column can be overwritten. The value stored there is the HTML produced by `adfToHTML`, which is what makes the round trip back through `htmlToADF` work. monday displays a long text column as plain text, so any markup Jira produced is visible as markup in the monday UI.

`items_page_by_column_values` is the monday equivalent of the JQL search used by the other sync templates: it finds the item carrying a given Jira issue key.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the monday item ID. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

On the monday board, create a text column for the Jira issue key, a long text column for the description, and a files column for attachments. Column IDs are visible through the monday API or in the column menu, and look like `text__1`, `long_text__1`, `files__1`.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key holding the monday item ID, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `1234567890` | Incoming and outgoing `MONDAY_BOARD_ID` flow variable | Replace with the numeric ID of the monday board that mirrors the Jira project. |
| `topics` | Outgoing `MONDAY_GROUP_ID` flow variable | Replace with the group ID on that board where new items should be created. |
| `text__1` | Incoming and outgoing `MONDAY_JIRA_KEY_COLUMN` flow variable | Replace with the ID of the monday text column holding the Jira issue key. |
| `long_text__1` | `MONDAY_DESCRIPTION_COLUMN` flow variable **and** the `Update Jira Issue Description` condition in the incoming flow | Replace with the ID of the monday long text column holding the description. The condition compares a literal column ID, so it must be changed in both places. |
| `files__1` | `MONDAY_FILES_COLUMN` flow variable **and** the `Get Monday Asset Public URL` condition in the incoming flow | Replace with the ID of the monday files column. The condition compares a literal column ID, so it must be changed in both places. |
| `12345678` | `MONDAY_IHUB_USER_ID` flow variable **and** the `Find Jira Issue by Monday Item ID` condition in the incoming flow | Replace with the monday user ID of the integration user whose API token iHub uses. The condition compares a literal user ID, so it must be changed in both places. This is the primary loop guard. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing update, comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular update, comment and attachment sync. |
| `2026-07` | Incoming and outgoing `MONDAY_API_VERSION` flow variable | The monday API version sent in the `API-Version` header. `2026-07` was the current stable version when this template was written; monday rotates versions quarterly and deprecates old ones, so check the [versioning docs](https://developer.monday.com/api-reference/docs/api-versioning) before enabling. |
| `MON` | Incoming `SPACE` flow variable | Replace with the Jira project key where monday items should create/find issues. |
| `Task` | Incoming `ISSUE_TYPE` flow variable | Replace if the target Jira project does not use `Task`. |

Note that `MONDAY_DESCRIPTION_COLUMN`, `MONDAY_FILES_COLUMN` and `MONDAY_IHUB_USER_ID` are declared as flow variables for visibility, but the conditions that branch on them compare literal strings. iHub conditions match a JSONPath against a static value, so a `{{_flow.X}}` reference in a condition value would be compared literally and never match, which for the user ID guard would fail open and allow sync loops. Both places must be edited together.

## Monday Webhook Setup

After the template is instantiated in iHub, copy the trigger URL from the `Monday Incoming Sync` custom webhook trigger. On the monday board, add an **Integrations -> Webhooks** recipe, or call the `create_webhook` mutation, pointing at that URL.

Subscribe to these four events:

| monday subscription event | Delivered `event.type` | Result in Jira |
| --- | --- | --- |
| `create_item` | `create_pulse` | Create a Jira issue |
| `change_column_value` | `update_column_value` | Update description or attach a file |
| `change_name` | `update_name` | Update the Jira summary |
| `create_update` | `create_update` | Add a Jira comment |

### The challenge handshake

When a webhook is registered, monday immediately POSTs

```json
{
  "challenge": "3eZbrw1aBm2rZgRNFdxV2595E9CY3gmdALWMmHkvFXO7tYXAYM8P"
}
```

to the target URL and expects the same value echoed back within seconds, or registration fails. This applies both to the board UI recipe and to the `create_webhook` mutation.

The incoming flow's trigger handles this with a custom webhook response template:

```json
"trigger": {
  "type": "CUSTOM_WEBHOOK_TRIGGER_TYPE",
  "config": {
    "response": {
      "mode": "custom",
      "customHeaders": [
        { "value": "application/json", "key": "Content-Type" }
      ],
      "customTemplate": "{{#challenge}}\n{\"challenge\":\"{{.}}\"}\n{{/challenge}}"
    }
  }
}
```

The section renders only when the incoming body has a `challenge` field, so registration handshakes are echoed and ordinary event deliveries get an empty body. If registration fails, check that this trigger config survived the import.

### Webhook payload

monday wraps every event in an `event` object. A `change_column_value` delivery looks like this:

```json
{
  "event": {
    "type": "update_column_value",
    "boardId": 1234567890,
    "pulseId": 9876543210,
    "pulseName": "Item name",
    "columnId": "long_text__1",
    "value": { "value": "New description text" },
    "previousValue": { "value": "Old description text" },
    "userId": 12345678
  }
}
```

**Legacy naming.** monday's API and webhook payloads still call items **pulses**. `event.pulseId` is the item ID and `event.pulseName` is the item name. Searching monday's docs for "itemId" will not find these. Note also that the subscription event name and the delivered `event.type` differ: subscribing to `create_item` delivers `create_pulse`.

The incoming flow reads the payload through tolerant scripted variables that unwrap a nested `payload` object, then the `event` object, and accept common key variants (`pulseId` / `pulse_id` / `itemId` / `item_id`, `textBody` / `body`, `value.files.0.assetId` and so on). `mondayEventType` normalises the delivered types to four canonical values: `create_item`, `change_column_value`, `change_name` and `create_update`. The payload above is the intended contract.

## Authentication

The manifest defines one monday.com credential: `custom-header-monday-api-token`, of type `CUSTOM_HEADER` with a single `Authorization` header.

monday API tokens go in the `Authorization` header with **no `Bearer` prefix** — the raw token is the entire header value. Generate one from **Developers -> My Access Tokens** in monday, or use a token belonging to a dedicated integration user. That user needs read and write access to the configured board, its columns and its updates.

Use a dedicated integration user rather than a personal token. Its monday user ID is what `MONDAY_IHUB_USER_ID` refers to, and the loop prevention below depends on every monday change iHub makes being attributable to that one user.

## Rate Limits

monday enforces a per-minute complexity budget and returns HTTP 429 with a `Retry-After` header when it is exhausted. Every monday-facing action carries a `ratelimit` block with `429` in `retryStatusCodes`, alongside 408 and the 5xx codes. Note that complexity is consumed per query, not per call, so a flow that syncs large boards can exhaust the budget faster than the request count suggests.

## Loop Prevention

Four mechanisms, layered:

1. **Author filtering.** The incoming flow's first action refuses any monday event whose `event.userId` matches the iHub integration user. The outgoing flow refuses Jira updates whose `user.accountId`, comments whose `comment.author.accountId`, and attachments whose `payload.author.accountId` match the iHub Jira account.
2. **Create only when unlinked.** The outgoing flow only creates a monday item when the Jira custom field is empty, and looks up an existing item by Jira key first. The incoming flow only creates a Jira issue when the JQL search returned nothing. Everything else routes to update.
3. **The link write-back is itself an event, and guard 1 catches it.** When the incoming flow writes the new Jira key into the monday text column, monday fires a `change_column_value` webhook for that write. Because iHub made it with the integration user's token, `event.userId` is the iHub user and the incoming flow's first action stops it. The mirror case is the outgoing flow storing the monday item ID on the Jira issue: that fires `avi:jira:updated:issue` with `user.accountId` set to the iHub Jira account, which the `Update Monday Item Name` and `Update Monday Item Description Column` conditions block.
4. **Issue properties.** Synced comments and attachments are recorded as Jira issue properties keyed `ihub-<monday-id>`, so a duplicate delivery of the same monday update or asset is not applied twice.

Guard 1 is the one that matters. If the monday token belongs to a real person who also works on the board, their own changes will be filtered out and the sync will silently drop them.

## Conflict Policy

Last write wins, with no merge. If the same field changes on both sides within one sync cycle, the change that arrives second overwrites the first. Neither flow reads the current value before writing, and there is no conflict detection or field-level timestamp comparison.

## Known Limitations

**Attachments are unverified against a live monday account.** The upload leg models monday's multipart GraphQL contract against `https://api.monday.com/v2/file`: a `query` form field carrying the `add_file_to_column` mutation with a `$file: File!` variable, a `map` form field of `{"image": "variables.file"}`, and the binary in an `image` part. iHub's multipart body model expresses this, and it mirrors monday's documented `curl` form exactly, but neither this nor the double-encoded `column_values` payload has been exercised against a real board. Test both legs on a scratch board before enabling in production.

**Asset URLs expire.** `assets { public_url }` returns a signed URL that expires after roughly an hour, so the incoming flow queries it and downloads it in the same run. If the download action is ever moved into a separate flow or a scheduled retry, the URL will already be dead.

**Description formatting.** The description is stored in a monday long text column as HTML, which round trips correctly back to Jira ADF but displays as raw markup in the monday UI. Switching to plain text would break the round trip.

**Redundant outgoing updates.** `Update Monday Item Name` and `Update Monday Item Description Column` fire on every `avi:jira:updated:issue` event, not only on summary and description changes, so an unrelated Jira field change re-sends both values. The writes are idempotent and monday does not emit a webhook when a column is set to its existing value, so this does not loop; it only costs complexity budget.

Out of scope for v1:

- **Status mapping.** monday statuses are per-board labels and Jira statuses move through workflow transitions. Mapping them needs a per-customer label-to-transition table.
- **Person and assignee mapping.** monday and Jira have separate user directories with no shared identifier.
- **Subitems.** Only top-level items are synced. `create_subitem` and the subitem events are normalised by the scripted variables but no flow acts on subitem-specific structure.
- **Multiple boards.** One Jira project maps to exactly one monday board. `MONDAY_BOARD_ID` is a single value in both flows.

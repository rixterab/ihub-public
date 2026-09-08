# Jira <-> ClickUp Task Sync

This template installs two iHub flows that keep Jira issues and ClickUp tasks linked in both directions. iHub is the master of the integration: it listens to Jira webhooks and to a ClickUp webhook (registered manually against iHub's custom webhook URL, see below), and drives every create, update and comment on both sides.

- `ClickUp Outgoing Sync`: Jira -> ClickUp, triggered by Jira webhooks.
- `ClickUp Incoming Sync`: ClickUp -> Jira, triggered by ClickUp's native webhook posting to an iHub custom webhook URL.

**This is a two-way sync.** Both directions create, update and comment; there is no primary system of record.

The sync stores the ClickUp task id in a Jira custom field, and the Jira issue key in a ClickUp custom field. Both are placeholders that must be replaced per customer — see [Required Customer Configuration](#required-customer-configuration).

## What The Template Does

From Jira to ClickUp (`ClickUp Outgoing Sync`):

- Creates a ClickUp task in the configured list when a Jira issue is created and no ClickUp task id is stored on it yet.
- Stores the created ClickUp task id back on the Jira issue.
- Writes the Jira issue key onto the new ClickUp task's linked-issue custom field.
- Updates the ClickUp task name and description when the Jira issue is updated.
- Adds Jira comments to the linked ClickUp task.
- Stores synced ClickUp comment ids as Jira issue properties using keys like `ihub-<clickup-comment-id>`, so the incoming flow does not sync them back.

From ClickUp to Jira (`ClickUp Incoming Sync`):

- Fetches the ClickUp task's current name, plain-text description and URL for every relevant event, because ClickUp's webhook payload carries only a task id and per-field history diffs, not the task itself.
- Searches Jira for an issue whose ClickUp task id custom field matches the incoming task.
- Creates a Jira issue when ClickUp sends a `taskCreated` event and no matching Jira issue exists, then writes the new Jira issue key back onto the ClickUp task's linked-issue custom field.
- Updates the matching Jira issue summary and description on a `taskUpdated` event.
- Adds ClickUp comments to the matching Jira issue on a `taskCommentPosted` event.
- Stores incoming ClickUp comment ids as Jira issue properties using keys like `ihub-<clickup-comment-id>` to avoid duplicate processing.

## ClickUp API Calls

| Action | Flow | Method | Endpoint |
| --- | --- | --- | --- |
| Create ClickUp Task | Outgoing | `POST` | `{{_flow.CLICKUP_API_URL}}/list/{{_flow.CLICKUP_LIST_ID}}/task` |
| Store Jira Issue Key on ClickUp Task | Outgoing | `POST` | `{{_flow.CLICKUP_API_URL}}/task/{taskId}/field/{{_flow.CLICKUP_JIRA_KEY_FIELD_ID}}` |
| Update ClickUp Task | Outgoing | `PUT` | `{{_flow.CLICKUP_API_URL}}/task/{taskId}` |
| Add Jira Comment to ClickUp Task | Outgoing | `POST` | `{{_flow.CLICKUP_API_URL}}/task/{taskId}/comment` |
| Get ClickUp Task | Incoming | `GET` | `{{_flow.CLICKUP_API_URL}}/task/{taskId}` |
| Store Jira Issue Key on ClickUp Task | Incoming | `POST` | `{{_flow.CLICKUP_API_URL}}/task/{taskId}/field/{{_flow.CLICKUP_JIRA_KEY_FIELD_ID}}` |

Jira-side calls (search, create, update, comment, issue properties) use the standard `{{baseUrl}}/rest/api/3/...` endpoints, exactly as in every other sync template in this repository.

Things that are easy to get wrong about the ClickUp API:

- **The `Authorization` header takes the raw API token, with no `Bearer` prefix.** ClickUp is like Linear in this respect, unlike GitHub or HubSpot.
- **Task and comment bodies are plain text or Markdown, never HTML and never Jira's ADF.** There is no ClickUp equivalent of `adfToHTML`/`htmlToADF` on ClickUp's side; iHub's helpers only translate to and from Jira's format.
- **Custom field values are written with a dedicated endpoint**, `POST /task/{task_id}/field/{field_id}`, with `{"value": "..."}` in the body — not as part of the task create/update payload.
- **ClickUp custom field ids are UUIDs**, not simple slugs, and are per-list (or per-folder/space if configured to apply everywhere). Read them with `GET /list/{list_id}/field`.
- **The task GET response's `url` field is the canonical browser link** (`https://app.clickup.com/t/<id>`); this template stores that value rather than reconstructing it.

## Body Format And Fidelity

Jira Cloud descriptions and comments are **ADF**. ClickUp task descriptions and comments are **plain text or Markdown**, and ClickUp does not render inline HTML in either field.

- **Jira -> ClickUp** (outgoing): the flow runs the Jira ADF description/comment through `{{{adfToHTML issue.fields.description}}}` / `{{{adfToHTML comment.body}}}` and inserts the result into ClickUp's plain-text `description` / `comment_text` field. **ClickUp will show the raw HTML tags** (`<p>...</p>`, `<strong>...</strong>`) as literal text, because it has no HTML renderer for these fields — this is the same class of trade-off `trello-card-from-jira` documents, except ClickUp does not even strip most inline tags the way Trello's Markdown renderer does, so the tag soup is more visible here than in Trello. This template makes that trade-off anyway, on the theory that a reader can still parse tagged text and the alternative (dropping the body) loses more information. If that is not acceptable for a given customer, replace the `adfToHTML` calls in `Create ClickUp Task`, `Update ClickUp Task` and `Add Jira Comment to ClickUp Task` with a scripted variable that walks the ADF tree to plain text, the way `linear-sync` does for Linear (which also renders Markdown, not HTML).
- **ClickUp -> Jira** (incoming): ClickUp's description/comment fields are plain text or Markdown, not HTML, so running them through `{{{htmlToADF x}}}` is not guaranteed safe in general — `htmlToADF` is documented as an HTML-to-ADF converter, and Markdown syntax is not HTML. This template makes a deliberate, documented choice: it feeds the plain ClickUp text directly into `{{{htmlToADF clickupTaskDescription}}}` / `{{{htmlToADF clickupCommentText}}}` anyway, on the same precedent `asana-sync` already sets in this repository for Asana's plain-text task notes. In practice this means: plain paragraphs come through fine, and any Markdown markers (`**bold**`, `- list item`) or literal angle brackets in the ClickUp text survive as visible characters in the resulting Jira ADF, exactly like the trade-off `linear-sync` documents for Markdown-into-ADF. If a customer's ClickUp usage relies heavily on Markdown formatting, consider stripping or converting Markdown to HTML before this point; this template does not attempt that.

Both directions prefix the synced body with a link back to the other system (`Synced from Jira issue [...]` / the ClickUp task URL captured in `clickupTaskUrl`), so a reader can always jump to the original regardless of formatting loss.

Free-text Jira and ClickUp values are interpolated with triple braces (`{{{ }}}`) so an `&` or apostrophe is not turned into an HTML entity. As in every other template here, a double quote inside a Jira summary or a ClickUp task name can still break the surrounding JSON body — this is a known, accepted, repository-wide trade-off, not specific to this template.

## Credential

The manifest defines one credential: `custom-header-clickup-api-key`, of type `CUSTOM_HEADER` with a single `Authorization` header.

Generate a personal API token in ClickUp under **Settings > Apps** (bottom of the left sidebar in the ClickUp web app), and paste it as the value of the `Authorization` header when configuring the credential in iHub. The token is sent **raw, with no `Bearer` prefix**:

```
Authorization: pk_1234567_ABCDEFGHIJKLMNOPQRSTUVWXYZ012345
```

Create the token under a dedicated ClickUp integration user rather than a personal account, so the loop guard on the ClickUp side (`CLICKUP_INTEGRATION_USER_ID`) has a stable user id to filter on and the token survives people leaving. That user needs access to the configured list (outgoing) and to whichever list/space the incoming webhook is registered against.

ClickUp also supports OAuth2 apps, which is the right choice for an application distributed to many workspaces. For a template configured by one customer against one workspace, a personal API token is the appropriate mechanism, matching the approach `linear-sync` and `trello-card-from-jira` take for their vendors.

## ClickUp Webhook Setup

ClickUp has **no UI panel for registering an arbitrary webhook** the way Slack or Datadog do. The webhook must be registered with a direct API call. After the template is instantiated in iHub:

1. Open the `ClickUp Incoming Sync` flow and copy its custom webhook trigger URL (the `CUSTOM_WEBHOOK_TRIGGER_TYPE` URL). Treat this URL as a credential — see [Security](#security) below.
2. Find the ClickUp team (workspace) id: `GET https://api.clickup.com/api/v2/team`, using the same personal API token as the `custom-header-clickup-api-key` credential, raw in the `Authorization` header.
3. Register the webhook by calling:

   ```
   POST https://api.clickup.com/api/v2/team/{team_id}/webhook
   Authorization: pk_1234567_ABCDEFGHIJKLMNOPQRSTUVWXYZ012345
   Content-Type: application/json

   {
     "endpoint": "<the iHub custom webhook trigger URL copied in step 1>",
     "events": ["taskCreated", "taskUpdated", "taskCommentPosted"],
     "list_id": "<the same CLICKUP_LIST_ID configured in the outgoing flow>"
   }
   ```

   Use `list_id`, `folder_id`, `space_id` or `team_id` scoping in the body depending on how broadly the sync should apply; the narrowest scope that still covers the target list is the safest choice. Omitting all of them subscribes the whole team, which also delivers events for tasks this integration is not meant to touch.
4. ClickUp responds with a `webhook` object containing `id` and a `secret`. **This template does not use the secret** — see [Security](#security). Keep the response for your records in case you need to update or delete the webhook later (`PUT`/`DELETE /api/v2/webhook/{webhook_id}`).
5. There is no challenge/handshake step; ClickUp starts delivering events immediately after the `POST` succeeds.

A ClickUp webhook delivery looks like this (per ClickUp's public webhook documentation; **this template's payload paths are the author's best understanding of that documentation, not verified against a real delivery** — confirm the exact paths in the flow log for your account before relying on the loop guard):

```json
{
  "webhook_id": "9a4b2c1d-0000-0000-0000-000000000000",
  "event": "taskUpdated",
  "task_id": "9hz9abc",
  "history_items": [
    {
      "id": "1000000000",
      "field": "status",
      "before": "to do",
      "after": "in progress",
      "user": { "id": 12345678, "username": "Ada Lovelace" }
    }
  ]
}
```

For `taskCommentPosted`, `history_items[0].comment` carries the comment content (an array of rich-text segments, each with a `text` field, or a plain string depending on ClickUp API version) and `history_items[0].id` is used by this template as the comment identity for the `ihub-<id>` issue-property dedup guard, since ClickUp's webhook documentation does not clearly name a separate top-level comment id field.

Event mapping used by the incoming flow:

| `event` | Result in Jira |
| --- | --- |
| `taskCreated` | Create a Jira issue and link it back with the ClickUp custom field |
| `taskUpdated` | Update the linked Jira issue summary and description |
| `taskCommentPosted` | Add a Jira comment on the linked issue |

Anything else ClickUp can send (`taskDeleted`, `taskMoved`, `taskTagUpdated`, and so on) is dropped, because `clickupSyncEvent` only normalizes those three values.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the ClickUp task id. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field id from their Jira instance.

Create a ClickUp custom field of type text on the target list to store the linked Jira issue key; read its UUID with `GET /list/{list_id}/field` after creating it.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://api.clickup.com/api/v2` | Both flows' `CLICKUP_API_URL` flow variable | Leave as-is unless ClickUp changes its API version. |
| `900100000000` | Outgoing `CLICKUP_LIST_ID` flow variable | Replace with the ClickUp list id that Jira issues are created into. |
| `00000000-0000-0000-0000-000000000000` | Both flows' `CLICKUP_JIRA_KEY_FIELD_ID` flow variable | Replace with the UUID of the ClickUp text custom field that stores the Jira issue key. Must be the same value in both flows. |
| `00000000` | Incoming `CLICKUP_INTEGRATION_USER_ID` flow variable | Replace with the numeric ClickUp user id of the integration user whose API token is used. This prevents circular sync. |
| `CU` | Incoming `SPACE` flow variable | Replace with the Jira project key where ClickUp-created tasks should create/find issues. |
| `Task` | Incoming `ISSUE_TYPE` flow variable | Replace if the target Jira project does not use `Task`. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field id, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing update and comment conditions | Replace with the Jira account id used by iHub/the integration user. This prevents circular sync. |

### Places To Update The Jira Custom Field

In `clickup_outgoing_sync.json`:

- `Create ClickUp Task` condition: checks `$.issue.fields.customfield_10153` is empty before creating.
- `Store ClickUp Task ID on Jira Issue` body: writes the ClickUp task id into `customfield_10153`.
- `Update ClickUp Task` and `Add Jira Comment to ClickUp Task`: read `issue.fields.customfield_10153` as the ClickUp task id and check it in their conditions.

In `clickup_incoming_sync.json`:

- `Search Jira Issue by ClickUp Task ID` JQL: searches `cf[10153]` for the incoming ClickUp task id.
- `Create Jira Issue from ClickUp Task` body: writes the incoming ClickUp task id into `customfield_10153`.

### Places To Update The ClickUp Custom Field

`CLICKUP_JIRA_KEY_FIELD_ID` is used in `Store Jira Issue Key on ClickUp Task` in **both** flow files — the outgoing flow writes the Jira key onto tasks it creates, the incoming flow writes it onto Jira issues created from ClickUp tasks. Both must point at the same ClickUp custom field.

## Security

Read this before enabling the incoming flow.

**iHub cannot verify ClickUp webhook signatures.** ClickUp returns a signing secret when a webhook is registered (see step 4 above), and it is documented to sign deliveries; this template does not check any signature, because iHub has no way to compute an HMAC over the raw request body inside a scripted variable or condition. There is no workaround in the flow engine.

What follows from that:

- **The only thing protecting the endpoint is that the iHub custom webhook URL is unique and unguessable. Treat it as a credential.** Do not commit it, do not paste it into tickets, chat messages or screenshots, and do not share it beyond the people configuring the integration. If it leaks, recreate the flow trigger to rotate the URL, then re-register the ClickUp webhook against the new URL (and delete the old ClickUp webhook registration).
- Because payloads are unauthenticated, anyone who learns the URL can post fabricated events and create or modify Jira issues through this flow. This template has **no reliable field to run a weak authenticity check against** — ClickUp's webhook payload (per its public documentation) does not clearly carry a team/workspace id alongside every event the way Linear's payload does, so unlike `linear-sync` this template cannot add an organization/team match as a cheap extra check. If your ClickUp webhook is scoped narrowly (to one list, as recommended in step 3 above), that scoping is the closest available substitute, but it is enforced by ClickUp at registration time, not verified by this flow.
- Replay is not detected. A captured request can be resent. The Jira issue property guard on comment ids limits duplicate comments, but a replayed `taskCreated` event for a task whose Jira link has since been removed could produce a second Jira issue.

## Loop Prevention

Every write one side of the sync makes is visible to the other side as a new event. Five guards keep that from looping:

1. **Jira side actor filter.** The outgoing update and comment actions require the acting Jira account to differ from the iHub integration account (`$.payload.user.accountId`, `$.comment.author.accountId`). Confirm those paths against a real Jira event in the flow log, since the actor field differs per Jira event type.
2. **ClickUp side actor filter.** The root action of the incoming flow requires the ClickUp actor id (`history_items[0].user.id`) to differ from the configured `CLICKUP_INTEGRATION_USER_ID`, so nothing iHub writes into ClickUp comes back to Jira.
3. **Create only when the link is absent.** The outgoing create action only runs when the Jira issue's ClickUp custom field is empty. The incoming flow searches Jira by the ClickUp task id first; only a miss (`EMPTY_ARRAY`) creates a new Jira issue, a hit (`NOT_EMPTY_ARRAY`) routes to update/comment.
4. **Jira issue properties.** Synced comment ids are stored as Jira issue properties keyed `ihub-<clickup-comment-id>`, and the incoming comment path reads the property first and only adds the comment when it is missing (a `404`).
5. **The link write-back is itself an update.** Storing the ClickUp task id in `customfield_10153` fires a Jira `issue_updated` webhook, caught by guard 1 (written by the integration account). Storing the Jira issue key onto the ClickUp custom field fires a ClickUp field-update event on that task, caught by guard 2 (written by the integration user). Verify both in the flow log, so the extra round trip does not create visible noise on every create.

Also check for loops *across* templates: installing this template alongside any other ClickUp<->Jira integration against the same list/project is a cycle risk this README cannot detect for you.

## Rate Limits

ClickUp's documented rate limit is 100 requests per minute per token on the Free/Unlimited plans, higher on Business/Enterprise plans. Outbound ClickUp-facing actions in both flows carry a conservative `ratelimit` block:

```json
"ratelimit": {
  "sleepTime": 500,
  "initialSleepTime": 5000,
  "maxRetries": 3,
  "burst": 20,
  "retryStatusCodes": [408, 429, 500, 502, 503, 504]
}
```

`maxRetries: 3` is used rather than `linear-sync`'s `maxRetries: 0`, because ClickUp signals rate limiting as a real HTTP `429` (unlike Linear's `200`/`400`-with-body pattern), so retrying on `429` and the standard `5xx` codes is safe here. Pure Jira write-back actions do not carry a `ratelimit` block, matching the rest of this repository.

## Conflict Policy

Last write wins, with no merge. If the same task/issue is edited on both sides at once, whichever event iHub processes last overwrites the other side's name/summary and description. There is no field-level merge, no conflict marker and no ordering guarantee between the two flows.

## Limitations

Out of scope for this version of the template:

- **Status/workflow mapping.** ClickUp custom statuses are per-list and per-workspace, with no stable identifier shared with Jira workflow statuses; this template does not attempt to map them in either direction. A future version would need a configurable status-name lookup table per list.
- **Assignee mapping.** ClickUp and Jira have separate user directories with no shared key.
- **Priority, tags, due dates, custom fields other than the link fields.**
- **Subtasks, checklists and dependencies.** ClickUp subtasks and checklist items are not synced; only the parent task's name, description and comments are.
- **Attachments/files.** Neither flow uploads or links file attachments in either direction. Adding this would need, on the ClickUp side, `POST /task/{task_id}/attachment` with a multipart body for outgoing files, and on the Jira side, downloading a ClickUp attachment URL and `POST`ing multipart to `{{baseUrl}}/rest/api/3/issue/{key}/attachments` for incoming files — the same two-step shape `asana-sync` already implements for Asana, which would be the template to copy.
- **Deletes.** ClickUp task deletions and Jira issue deletions are not synced.
- **Webhook signature verification and replay protection.** See [Security](#security).
- **Payload shape confidence.** The `history_items` / comment payload paths this template reads are the author's best understanding of ClickUp's public webhook documentation, not confirmed against a real delivery. Confirm `history_items[0].user.id`, `history_items[0].id` (used as the comment identity) and the shape of `history_items[0].comment` in the flow log for your ClickUp account before trusting the loop guard or the comment dedup guard in production, and adjust the scripted variables in `Get ClickUp Task` if they differ.
- **The ClickUp icon shipped with this template is a placeholder**, not ClickUp's official logo mark. It is a plain rounded square using ClickUp's brand purple-to-pink gradient with a simple chevron glyph, built without copying any official asset. Replace `clickup.svg` with the official ClickUp logo before distributing this template outside of testing, per this repository's rule against inventing logo path data.

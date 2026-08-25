# Jira <-> Asana Task Sync

This template installs two iHub flows that keep Jira issues and Asana Tasks linked in both directions. iHub is the master of the sync: it listens to Asana's native webhooks and is responsible for creating the matching Jira issue when a new Asana task arrives.

- `Asana Outgoing Sync`: Jira -> Asana, triggered by Jira webhooks.
- `Asana Incoming Sync`: Asana -> Jira, triggered by Asana's native webhooks posting to an iHub custom webhook URL.
- `Asana Register Webhook`: a one-time setup flow that registers the Asana Incoming Sync trigger URL as an Asana webhook.

The sync stores the Asana Task GID in a Jira custom field. That field is the link key used by both flows to find the matching record.

## What The Template Syncs

From Jira to Asana:

- Creates an Asana Task when a Jira issue is created and the Jira issue does not already have an Asana Task GID.
- Stores the created Asana Task GID back on the Jira issue.
- Updates the Asana Task `notes` when the Jira issue is updated.
- Adds Jira comments to the linked Asana Task as Stories.
- Uploads Jira attachments to the linked Asana Task using the Asana Attachments API.
- Stores synced Asana story and attachment GIDs as Jira issue properties using keys like `ihub-<asana-gid>` to avoid duplicate processing.

From Asana to Jira:

- Searches Jira for an issue whose Asana Task GID custom field matches the incoming Task GID.
- Creates a Jira issue when Asana sends a task-added webhook event and no matching Jira issue exists.
- Adds Asana Story comments to the matching Jira issue.
- Downloads Asana attachments and uploads them to the matching Jira issue.
- Stores incoming Asana story and attachment GIDs as Jira issue properties using keys like `ihub-<asana-gid>`.

Asana's webhook events only carry resource GIDs and an action, not the actual content. The incoming flow makes an authenticated Asana API call for every event to fetch the task name/notes, story text, or attachment name/download URL before writing anything to Jira.

## Asana API Calls

The outgoing flow uses OAuth2 and the Asana REST API:

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Task | `POST` | `{{_flow.ASANA_URL}}/tasks` |
| Comment on Task | `POST` | `{{_flow.ASANA_URL}}/tasks/{{issue.fields.customfield_10200}}/stories` |
| Update Task Notes | `PUT` | `{{_flow.ASANA_URL}}/tasks/{{issue.fields.customfield_10200}}` |
| Upload Attachment | `POST` | `{{_flow.ASANA_URL}}/attachments` |

The incoming flow uses the same OAuth2 credential to enrich webhook events:

| Action | Method | Endpoint |
| --- | --- | --- |
| Get Task | `GET` | `{{_flow.ASANA_URL}}/tasks/{{asanaTaskGid}}` |
| Get Story | `GET` | `{{_flow.ASANA_URL}}/stories/{{asanaStoryGid}}` |
| Get Attachment | `GET` | `{{_flow.ASANA_URL}}/attachments/{{asanaAttachmentGid}}` |

Attachment downloads are a two-step process: the Get Attachment call returns a temporary, pre-signed `download_url`, and a second unauthenticated `GET` against that URL retrieves the file bytes.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Asana Task GID. The template currently uses `customfield_10200` / `cf[10200]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://app.asana.com/api/1.0` | Incoming and outgoing `ASANA_URL` flow variable | Replace only if the customer uses a proxied or region-specific Asana API base URL. |
| `0000000000000000` | Outgoing `ASANA_WORKSPACE_GID` flow variable | Replace with the customer's Asana workspace GID. |
| `0000000000000000` | Outgoing `ASANA_PROJECT_GID` flow variable | Replace with the Asana project GID that newly created tasks are added to. |
| `0000000000000000` | Register Webhook `ASANA_PROJECT_GID` flow variable | Replace with the same Asana project GID; this is the project the incoming webhook subscribes to. |
| _(empty)_ | Register Webhook `ASANA_TARGET_URL` flow variable | Set to the `Asana Incoming Sync` custom webhook trigger URL after that flow is created. |
| `AS` | Incoming `SPACE` flow variable | Replace with the Jira project key where Asana-created tasks should create/find issues. |
| `Task` | Incoming `ISSUE_TYPE` flow variable | Replace if the target Jira project does not use `Task`. |
| `customfield_10200` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10200]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular comment and attachment sync. |

## Asana Webhook Setup

After the template is instantiated in iHub:

1. Open the `Asana Incoming Sync` flow and copy its custom webhook trigger URL.
2. Open the `Asana Register Webhook` flow (disabled by default, it is only meant to be run once) and set its flow variables:
   - `ASANA_PROJECT_GID`: the Asana project GID to subscribe to (use the same project `Asana Outgoing Sync` creates tasks in).
   - `ASANA_TARGET_URL`: the trigger URL copied in step 1.
3. Run the `Register Asana Webhook` action once via Test Flow. It calls `POST {{_flow.ASANA_URL}}/webhooks` with `resource` and `target` set from those flow variables.

Asana requires the target URL to complete a one-time handshake as part of that call: immediately after `POST /webhooks`, Asana sends an empty request to the target URL with an `X-Hook-Secret` header, and expects a `200` response that echoes the same header back within a few seconds, or the subscription is rejected. The `Asana Incoming Sync` trigger is configured with a custom response (`trigger.config.response`) that echoes the `X-Hook-Secret` request header back as a response header automatically, so no manual handshake step is needed:

```json
"trigger": {
  "type": "CUSTOM_WEBHOOK_TRIGGER_TYPE",
  "config": {
    "response": {
      "mode": "custom",
      "customHeaders": [
        { "key": "Content-Type", "value": "application/json" },
        { "key": "X-Hook-Secret", "value": "{{headers.x-hook-secret}}" }
      ],
      "customTemplate": "{}"
    }
  }
}
```

On regular event deliveries this same response is returned (Asana ignores the body and the empty `X-Hook-Secret` header on those requests, since the handshake is already complete).

Once subscribed, Asana delivers batched webhook payloads shaped like:

```json
{
  "events": [
    {
      "resource": { "gid": "1234567890", "resource_type": "task" },
      "parent": null,
      "action": "added",
      "user": { "gid": "..." },
      "created_at": "2026-08-24T00:00:00.000Z"
    }
  ]
}
```

The incoming flow iterates over `events` and classifies each one:

- `resource.resource_type = "task"`, `action = "added"` -> `task.created`
- `resource.resource_type = "story"`, `action = "added"` -> `task.commented` (task GID comes from `parent.gid`)
- `resource.resource_type = "attachment"`, `action = "added"` -> `task.attachment.created` (task GID comes from `parent.gid`)

## OAuth2

The manifest defines one Asana OAuth2 credential: `oauth2-asana`. It uses the `default` scope (full account access) and is used by every Asana REST call in both flows.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. Incoming Asana events also check Jira issue properties before creating comments or attachments. This prevents events created by one side of the sync from being immediately sent back to the source side.

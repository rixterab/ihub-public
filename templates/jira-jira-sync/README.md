# Jira <-> Jira Issue Sync

This template installs two iHub flows that keep issues in sync between two Jira Cloud instances. iHub is the master of the integration: it listens to webhooks from the instance it is installed in and to webhooks posted from the other instance, and drives every create and update on both sides.

The two sides are not symmetric. iHub is installed in one instance only:

- The **local** instance is where iHub is installed. It is reached through `{{baseUrl}}` with the `ATLASSIAN_TOKEN` credential reference, and its events arrive on the Atlassian webhook trigger.
- The **remote** instance is just another REST API. It is reached through the `REMOTE_URL` flow variable with a `BASIC_AUTH` credential, and its events arrive on an iHub custom webhook URL.

The two flows:

- `Jira Outgoing Sync`: local -> remote, triggered by local Jira webhooks (`ATLASSIAN_WEBHOOK_TRIGGER_TYPE`).
- `Jira Incoming Sync`: remote -> local, triggered by the remote instance posting standard Jira webhook bodies to an iHub custom webhook URL (`CUSTOM_WEBHOOK_TRIGGER_TYPE`).

Both flows use the Jira Cloud REST API v3 on both sides. Search uses `/rest/api/3/search/jql`; the old `/rest/api/3/search` endpoint no longer exists on Cloud. Server and Data Center are not supported.

Each instance stores the other instance's issue key in a custom field. Those two fields are the link key used by both flows to find the matching issue. They are **two different fields with two different ids**: `customfield_10153` on the local instance and `customfield_10200` on the remote instance. Custom field ids are per instance and will almost never match, so both must be replaced separately.

## What The Template Syncs

From local to remote:

- Searches the remote instance for an issue whose link field `customfield_10200` already holds the local issue key.
- Creates a remote issue when a local issue is created, the local link field is empty and the remote search returned nothing.
- Links the local issue to an existing remote mirror instead of creating a duplicate when the remote search does return a match.
- Stores the created remote issue key back in the local link field `customfield_10153`.
- Updates the remote issue summary and description when the local issue is updated.
- Adds local comments to the linked remote issue.
- Downloads local attachments and uploads them to the linked remote issue.
- Stores synced comment and attachment ids as local Jira issue properties using keys like `ihub-<id>` to avoid duplicate processing.

From remote to local:

- Searches the local instance for an issue whose link field `customfield_10153` matches the incoming remote issue key.
- Creates a local issue on a `jira:issue_created` event when no matching local issue exists and the remote link field is empty.
- Writes the new local issue key into the remote link field `customfield_10200`.
- Updates the matching local issue summary and description on a `jira:issue_updated` event.
- Adds remote comments to the matching local issue on a `comment_created` event.
- Downloads remote attachments and uploads them to the matching local issue.
- Stores incoming remote comment and attachment ids as local Jira issue properties using keys like `ihub-<id>`.

The link value stored on both sides is the issue **key**, because it is human readable and can be used directly in REST URLs such as `/rest/api/3/issue/{key}`. If projects may be renamed or issues moved between projects, the issue **id** is the more stable choice: it never changes, while the key does. To switch, store `{{issue.id}}` instead of `{{issue.key}}` in the create bodies and in the write-back actions, and search on the id value instead.

## Jira API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Find local issue | `POST` | `{{baseUrl}}/rest/api/3/search/jql` |
| Find remote issue | `POST` | `{{_flow.REMOTE_URL}}/rest/api/3/search/jql` |
| Create local issue | `POST` | `{{baseUrl}}/rest/api/3/issue` |
| Update local issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}` |
| Comment local issue | `POST` | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}/comment` |
| Upload local attachment | `POST` | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}/attachments` |
| Create remote issue | `POST` | `{{_flow.REMOTE_URL}}/rest/api/3/issue` |
| Update remote issue | `PUT` | `{{_flow.REMOTE_URL}}/rest/api/3/issue/{{issue.fields.customfield_10153}}` |
| Comment remote issue | `POST` | `{{_flow.REMOTE_URL}}/rest/api/3/issue/{{issue.fields.customfield_10153}}/comment` |
| Upload remote attachment | `POST` | `{{_flow.REMOTE_URL}}/rest/api/3/issue/{{issue.fields.customfield_10153}}/attachments` |
| Download attachment | `GET` | The attachment `content` URL from `issue.fields.attachment[]` |

Attachments are download-then-upload in both directions. The source attachment `content` URL is fetched with the credential of the instance that owns it, stored as an iHub input file, then posted as multipart to `/rest/api/3/issue/{key}/attachments` on the target instance with the `X-Atlassian-Token: no-check` header.

## Description And Comment Bodies

Both instances are Jira Cloud, so both speak ADF (Atlassian Document Format). The template passes the ADF document through unchanged with the `{{{toJSON ...}}}` helper rather than round-tripping it through HTML:

```
"description": {{{toJSON issue.fields.description}}}
"body": {{{toJSON comment.body}}}
```

Do not replace these with an `{{{adfToHTML ...}}}` / `{{{htmlToADF ...}}}` pair copied from one of the other sync templates in this repository. Those templates need the conversion because the system on the other side is not ADF-aware. Here the conversion is pure loss: tables, panels, media nodes, inline mentions, status lozenges and expand macros do not survive the round trip.

On the incoming side the ADF document arrives inside the webhook body and is read by the `remoteDescription` and `remoteCommentBody` scripted variables, which return the ADF object as-is. When the source field is empty, they return an empty ADF document (`{ "type": "doc", "version": 1, "content": [] }`) so the request body stays valid.

Known fidelity limitation: media nodes reference attachment ids that are local to the source instance. The attachment binary is copied to the target issue, but an inline image embedded in a description or comment will point at an id that does not exist on the target and will render as a broken media node. The file itself is still present in the target issue's attachment list.

## Loop Prevention

Both instances emit the same kind of webhook, so an unguarded build loops forever: A creates, B creates, B's create webhook fires, that creates in A, and so on. The template layers four guards.

**1. Author filter, per instance.** Every write action skips events authored by the iHub integration account. This needs two different account ids, because the integration user has a different `accountId` in each instance:

- Outgoing checks `$.issue.fields.creator.accountId`, `$.user.accountId`, `$.comment.author.accountId` and `$.payload.author.accountId` against the **local** iHub account id.
- Incoming checks the `remoteAuthorAccountId` and `remoteCommentAuthorAccountId` scripted variables against the **remote** iHub account id. The incoming author check also sits on the root search action, so a remote event authored by the remote iHub user stops the whole flow before any write happens.

**2. Link-field check on create.** Outgoing only creates a remote issue when `$.issue.fields.customfield_10153` is empty and a JQL search of the remote project on `cf[10200]` returns no issue already holding the local key; incoming only creates a local issue when `remoteLinkFieldValue` (`customfield_10200`) is empty and a JQL search of the local project on `cf[10153]` returned no issues. If the link field is populated, the issue is already a mirror and the event routes to the update action instead.

Both create actions set the other side's link field **in the create request body**, not afterwards. That means the mirror issue's own `jira:issue_created` webhook already carries a populated link field, so this guard holds on the very first event rather than racing a later write.

**3. The write-back is itself an update event.** When a flow writes the other side's key into the link field, that fires `jira:issue_updated` on the instance that was written to. This is covered twice:

- Outgoing action `Store Remote Issue Key on Local Issue` writes with the local iHub credential, so the resulting local `jira:issue_updated` carries `$.user.accountId` equal to the local iHub account id and is blocked by guard 1 on `Update Remote Issue`.
- Incoming action `Store Local Issue Key on Remote Issue` writes with the remote `BASIC_AUTH` credential, so the resulting remote `jira:issue_updated` carries the remote iHub account id and is blocked by guard 1 on the root search action.

As a second layer that does not depend on the author being populated, both update actions also evaluate a `localLinkFieldOnlyChange` / `remoteLinkFieldOnlyChange` scripted variable. It returns `true` when every entry in `changelog.items[]` is the link field, and the update action is skipped in that case. A pure link-field write can therefore never propagate, even if the author check fails.

**4. Issue properties for comments and attachments.** Before copying a comment or an attachment, both flows `GET` a Jira issue property named `ihub-<id>` and only proceed on a `404`. After a successful copy the property is written. This makes repeated webhooks idempotent, which matters for attachments in particular: attachments are detected from `jira:issue_updated` and the flow iterates the full `issue.fields.attachment[]` array, so without this guard every later update would re-upload every existing attachment.

The property key namespaces differ per direction: the outgoing flow keys on the **local** attachment or comment id, the incoming flow keys on the **remote** id. Both are stored on the local issue.

Conflict policy is **last write wins, with no merge**. If the same field is edited on both instances at the same time, whichever webhook iHub processes last overwrites the other side. There is no field-level merge, no three-way diff and no conflict marker.

## Incoming Payload

The remote instance is configured to POST the **standard Jira Cloud webhook body**; there is no custom payload contract. The incoming flow reads it with tolerant scripted variables that accept the native shape and a nested `payload` wrapper:

| Scripted variable | Read from |
| --- | --- |
| `remoteEventType` | `webhookEvent`, `event`, `eventType`, `issue_event_type_name` |
| `remoteIssueKey` | `issue.key`, `issueKey`, `key` |
| `remoteIssueId` | `issue.id`, `issueId`, `id` |
| `remoteSummary` | `issue.fields.summary`, `fields.summary`, `summary` |
| `remoteDescription` | `issue.fields.description`, `fields.description`, `description` |
| `remoteLinkFieldValue` | `issue.fields.customfield_10200` |
| `remoteAuthorAccountId` | `user.accountId`, `issue.fields.creator.accountId` |
| `remoteCommentId` | `comment.id`, `commentId` |
| `remoteCommentBody` | `comment.body`, `commentBody`, `body` |
| `remoteCommentAuthorAccountId` | `comment.author.accountId`, `comment.updateAuthor.accountId` |
| `remoteHasAttachmentChange` | `changelog.items[].field == "Attachment"` |
| `remoteAttachments` | `issue.fields.attachment` |

Event mapping:

| `webhookEvent` | Result |
| --- | --- |
| `jira:issue_created` | Create the issue on the other side |
| `jira:issue_updated` | Update summary and description |
| `comment_created` | Add the comment |
| `jira:issue_updated` with a changelog entry for `Attachment` | Copy attachments |

Jira has no dedicated attachment webhook event. Attachments are detected from a `jira:issue_updated` event whose `changelog.items[]` contains an entry with `field` equal to `Attachment`, after which the flow iterates `issue.fields.attachment[]` and copies each entry that has not been synced yet.

A representative incoming body for a comment event:

```json
{
  "webhookEvent": "comment_created",
  "issue": {
    "id": "10042",
    "key": "REMOTE-17",
    "fields": {
      "summary": "Issue summary",
      "customfield_10200": "LOCAL-9"
    }
  },
  "comment": {
    "id": "10501",
    "author": { "accountId": "712020:1f4c8d90-5b73-42ae-9c61-08d7e3b5a2f4" },
    "body": { "type": "doc", "version": 1, "content": [] }
  }
}
```

## Remote Webhook Setup

After the template is instantiated in iHub, copy the trigger URL from the `Jira Incoming Sync` custom webhook trigger, then configure the remote instance to post to it.

Using a native webhook:

1. In the remote instance go to **Settings > System > WebHooks** and choose **Create a WebHook**.
2. Set the **URL** to the iHub custom webhook trigger URL.
3. Set the JQL scope to the synced project, for example `project = REMOTE`.
4. Enable the events **Issue: created**, **Issue: updated** and **Comment: created**.
5. Leave the payload as the default Jira body. Do not enable payload exclusion, the flow needs `issue.fields` and `changelog`.
6. Save and enable the webhook.

Using an Automation rule instead:

1. Create a rule with the triggers **Issue created**, **Issue updated** and **Issue commented**.
2. Add a **Send web request** action pointing at the iHub custom webhook trigger URL, method `POST`, with the webhook body set to **Issue data (automation format)** replaced by a custom body matching the native shape above, or keep the native webhook approach which needs no body mapping.
3. Exclude the remote iHub integration user in the rule condition as an extra safety net.

The local instance needs no manual webhook configuration; the Atlassian webhook trigger is wired by iHub when the flow is enabled.

## Required Customer Configuration

Create a single-line text custom field in **each** instance to hold the other instance's issue key. The template uses `customfield_10153` on the local instance and `customfield_10200` on the remote instance. These are placeholders; the ids in the customer's two instances will differ from each other and from these values.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `customfield_10153` | Both flow JSON files | Replace with the **local** instance link field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric id of the **local** link field, for example `cf[12345]`. |
| `customfield_10200` | Both flow JSON files | Replace with the **remote** instance link field key. This is a different field from the local one. |
| `cf[10200]` | Outgoing flow JQL search, and the incoming `remoteLinkFieldValue` / `remoteLinkFieldOnlyChange` scripted variables | Replace with the numeric id of the **remote** link field. |
| `https://your-remote-instance.atlassian.net` | `REMOTE_URL` flow variable in both flows | Replace with the remote Jira Cloud base URL, for example `https://acme.atlassian.net`. |
| `LOCAL` | `SPACE` flow variable in both flows | Replace with the project key in the local instance that holds the synced issues. |
| `REMOTE` | `REMOTE_PROJECT` flow variable in both flows | Replace with the project key in the remote instance that holds the synced issues. |
| `Task` | `ISSUE_TYPE` and `REMOTE_ISSUE_TYPE` flow variables | Replace with issue type names that exist in each project. Issue type names differ per instance, which is why these are two separate variables. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | `LOCAL_IHUB_ACCOUNT_ID` flow variable **and** the outgoing action conditions | Replace with the `accountId` of the iHub integration user in the **local** instance, in both places. |
| `712020:1f4c8d90-5b73-42ae-9c61-08d7e3b5a2f4` | `REMOTE_IHUB_ACCOUNT_ID` flow variable **and** the incoming action conditions | Replace with the `accountId` of the iHub integration user in the **remote** instance, in both places. These are two different accounts. |
| `integration@your-company.com` | `manifest.json` credential definition | Replace with the Atlassian account email used for the remote API token, before importing or by editing the credential after import. |

The account ids appear both as flow variables and as literal values inside action conditions, because iHub condition values are compared literally and are not templated. The flow variables are the documented reference value; the conditions are what actually enforces the guard. Both must carry the same value or loop prevention will not work.

To find an `accountId`, call `GET /rest/api/3/myself` in each instance while authenticated as that instance's integration user.

## Credentials

The manifest defines one credential: `basic-auth-remote-jira`, of type `BASIC_AUTH`. It is used by every call to the remote instance.

- **Username**: the Atlassian account email of the integration user in the remote instance.
- **Password**: an Atlassian API token created by that user at `https://id.atlassian.com/manage-profile/security/api-tokens`.

Calls to the local instance use the `ATLASSIAN_TOKEN` credential reference `ATLASSIAN_AUTH_TOKEN` and need no configuration.

The remote integration user needs Browse Projects, Create Issues, Edit Issues, Add Comments and Create Attachments permissions in the remote project, and the local integration user needs the same in the local project. Both also need the ability to edit their instance's link custom field.

## Known Limitations

Out of scope for v1:

- **Status sync.** Workflows differ per instance, so a status name on one side may not exist on the other. Implementing it means calling `GET /rest/api/3/issue/{key}/transitions` on the target, matching the desired status to an available transition id, and `POST`ing to the same endpoint. A status mapping table between the two workflows is required.
- **Assignee and reporter mapping.** `accountId` values are per instance; the same person has different ids in each. This needs a user mapping table, or a lookup by email through `/rest/api/3/user/search`, which is often restricted by Atlassian privacy settings.
- **Sprints, epics and issue links.** Sprint and epic references point at board and issue ids that do not exist on the other instance. Issue links would require both linked issues to already be mirrored.
- **Priority, labels, components and other fields.** Only summary and description are synced.
- **Deletes.** Deleting an issue, comment or attachment on one side does not remove it on the other.
- **Inline media in ADF.** See the fidelity limitation described under Description And Comment Bodies.
- **Conflicts.** Last write wins, with no merge.

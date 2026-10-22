# Jira <-> Jira Issue Sync

This template installs two iHub flows that keep issues in sync **two ways** between two Jira Cloud instances. iHub is the master of the integration: it listens to webhooks from the instance it is installed in and to webhooks posted from the other instance, and drives every create and update on both sides.

The two sides are not symmetric. iHub is installed in one instance only:

- The **local** instance is where iHub is installed. It is reached through `{{baseUrl}}` with the `ATLASSIAN_TOKEN` credential reference, and its events arrive on the Atlassian webhook trigger.
- The **remote** instance is just another REST API. It is reached through the `REMOTE_URL` flow variable with a `BASIC_AUTH` credential, and its events arrive on an iHub custom webhook URL.

The two flows:

- `Jira Outgoing Sync`: local -> remote, triggered by local Jira webhooks (`ATLASSIAN_WEBHOOK_TRIGGER_TYPE`).
- `Jira Incoming Sync`: remote -> local, triggered by the remote instance posting its standard Jira webhook body to an iHub custom webhook URL (`CUSTOM_WEBHOOK_TRIGGER_TYPE`).

Both flows use the Jira Cloud REST API v3 on both sides. Search uses `/rest/api/3/search/jql`; the old `/rest/api/3/search` endpoint no longer exists on Cloud. Server and Data Center are not supported.

Each instance stores the other instance's issue key in a custom field. Those two fields are the link key used by both flows to find the matching issue. They are **two different fields with two different ids**: `customfield_10153` on the local instance and `customfield_10200` on the remote instance. Custom field ids are per instance and will almost never match, so both must be replaced separately.

## What The Template Does

Outgoing, local -> remote, in execution order:

1. On a local issue created event, when the local link field `customfield_10153` is empty and the creator is not the local integration user, search the remote project for an issue whose link field `customfield_10200` already holds the local issue key.
2. If the search finds nothing, create the remote issue (summary, description and the local key in `customfield_10200`) and write the new remote key back into the local `customfield_10153`.
3. If the search finds a match, link the local issue to that existing remote issue instead of creating a duplicate.
4. On a local issue updated event, update the linked remote issue's summary and description.
5. On a local comment event, check the local issue property `ihub-local-<comment id>`; if it does not exist, post the comment to the remote issue and write the property.
6. On every local event for a linked issue, iterate `issue.fields.attachment[]`, skip attachments uploaded by the local integration user, and for each attachment without an `ihub-local-<attachment id>` property download it from the local instance, upload it to the remote issue and write the property.

Incoming, remote -> local, in execution order:

1. Skip the whole event when it was triggered by the remote integration user (`$.user.accountId`). Otherwise search the local project for an issue whose `customfield_10153` holds the incoming remote issue key.
2. On `jira:issue_created`, when the search found nothing and the remote link field `customfield_10200` is empty, create the local issue (with the remote key in `customfield_10153`) and write the new local key back into the remote `customfield_10200`.
3. On `jira:issue_updated` with a non-empty changelog that is not only the link field, update the local issue's summary and description.
4. On `comment_created`, when the comment author is not the remote integration user and the local issue property `ihub-remote-<comment id>` does not exist, add the comment to the local issue and write the property.
5. On `jira:issue_updated` whose changelog contains an `Attachment` entry, iterate `issue.fields.attachment[]`, skip attachments uploaded by the remote integration user, and for each attachment without an `ihub-remote-<attachment id>` property download it from the remote instance, upload it to the local issue and write the property.

The link value stored on both sides is the issue **key**, because it is human readable and can be used directly in REST URLs such as `/rest/api/3/issue/{key}`. If projects may be renamed or issues moved between projects, the issue **id** is the more stable choice: it never changes, while the key does. To switch, store `{{issue.id}}` instead of `{{issue.key}}` in the create bodies and in the write-back actions, and search on the id value instead.

## API Calls

| Flow     | Action                                  | Method | Endpoint                                                                                   |
| -------- | --------------------------------------- | ------ | ------------------------------------------------------------------------------------------ |
| Outgoing | Find Remote Issue by Local Issue Key    | `POST` | `{{_flow.REMOTE_URL}}/rest/api/3/search/jql`                                               |
| Outgoing | Create Issue in Remote Jira             | `POST` | `{{_flow.REMOTE_URL}}/rest/api/3/issue`                                                    |
| Outgoing | Store Remote Issue Key on Local Issue   | `PUT`  | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}`                                               |
| Outgoing | Link Existing Remote Issue              | `PUT`  | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}`                                               |
| Outgoing | Update Remote Issue                     | `PUT`  | `{{_flow.REMOTE_URL}}/rest/api/3/issue/{{issue.fields.customfield_10153}}`                 |
| Outgoing | Get Local Comment ID Property           | `GET`  | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/properties/ihub-local-{{comment.id}}`          |
| Outgoing | Comment Remote Issue                    | `POST` | `{{_flow.REMOTE_URL}}/rest/api/3/issue/{{issue.fields.customfield_10153}}/comment`         |
| Outgoing | Store Outgoing Comment ID               | `PUT`  | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/properties/ihub-local-{{comment.id}}`          |
| Outgoing | Get Local Attachment ID Property        | `GET`  | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/properties/ihub-local-{{id}}`                  |
| Outgoing | Download Local Attachment               | `GET`  | The attachment `content` URL                                                               |
| Outgoing | Upload Attachment to Remote Issue       | `POST` | `{{_flow.REMOTE_URL}}/rest/api/3/issue/{{issue.fields.customfield_10153}}/attachments`     |
| Outgoing | Store Outgoing Attachment ID            | `PUT`  | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/properties/ihub-local-{{id}}`                  |
| Incoming | Find Local Issue by Remote Issue Key    | `POST` | `{{baseUrl}}/rest/api/3/search/jql`                                                        |
| Incoming | Create Local Issue                      | `POST` | `{{baseUrl}}/rest/api/3/issue`                                                             |
| Incoming | Store Local Issue Key on Remote Issue   | `PUT`  | `{{_flow.REMOTE_URL}}/rest/api/3/issue/{{issue.key}}`                                      |
| Incoming | Update Local Issue                      | `PUT`  | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}`                                          |
| Incoming | Get Remote Comment ID Property          | `GET`  | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}/properties/ihub-remote-{{comment.id}}`    |
| Incoming | Add Remote Comment to Local Issue       | `POST` | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}/comment`                                  |
| Incoming | Store Incoming Remote Comment ID        | `PUT`  | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}/properties/ihub-remote-{{comment.id}}`    |
| Incoming | Get Remote Attachment ID Property       | `GET`  | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}/properties/ihub-remote-{{id}}`            |
| Incoming | Download Remote Attachment              | `GET`  | The attachment `content` URL                                                               |
| Incoming | Add Remote Attachment to Local Issue    | `POST` | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}/attachments`                              |
| Incoming | Store Incoming Remote Attachment ID     | `PUT`  | `{{baseUrl}}/rest/api/3/issue/{{_flow.issueKey}}/properties/ihub-remote-{{id}}`            |

Attachments are download-then-upload in both directions. The source attachment `content` URL is fetched with the credential of the instance that owns it, stored as an iHub input file, then posted as multipart to `/rest/api/3/issue/{key}/attachments` on the target instance with the `X-Atlassian-Token: no-check` header. Inside the attachment iteration, `{{id}}`, `{{filename}}`, `{{{content}}}` and `$.payload.author.accountId` refer to the current attachment, the same way the other sync templates in this repository read iterated attachments.

Every call to the remote instance carries the repository's standard `ratelimit` block, because the remote Jira enforces its own rate limits independently of the local one. Local Jira calls do not.

## Body Format And Fidelity

**Outgoing.** The iHub Atlassian webhook trigger delivers `issue.fields.description` and `comment.body` as ADF (Atlassian Document Format), and the remote instance's v3 API accepts ADF, so the document is passed through unchanged:

```
"description": {{{toJSON issue.fields.description}}}
"body": {{{toJSON comment.body}}}
```

Do not replace these with an `{{{adfToHTML ...}}}` / `{{{htmlToADF ...}}}` pair copied from one of the other sync templates in this repository. Those templates need the conversion because the system on the other side is not ADF-aware. Here the conversion is pure loss: tables, panels, media nodes, inline mentions, status lozenges and expand macros do not survive the round trip.

**Incoming.** The remote instance's admin webhook does not deliver a consistent format. `issue.fields.description` and `comment.body` arrive as a wiki markup string in the classic webhook representation, but can arrive as ADF depending on how the content was written. The local v3 API accepts only ADF. This is the one place the incoming flow needs a script, because no Handlebars helper converts wiki markup to ADF:

- `remoteDescription` and `remoteCommentBody` pass an ADF object through unchanged. A string becomes an ADF document with one plain-text paragraph per non-empty line. An empty or missing value becomes an empty ADF document, so the request body stays valid.
- Both return a JSON **string**, which is inserted unquoted: `"description": {{{remoteDescription}}}`.

What is lost on the incoming path when the remote sends wiki markup: all formatting. `*bold*`, `{code}` blocks, tables and links arrive as their literal markup characters. If your remote instance turns out to send wiki markup every time, a higher-fidelity alternative is to post incoming descriptions and comments to the local **v2** API (`/rest/api/2/issue/...`), which accepts wiki markup and lets Jira do the conversion, with `{{{toJSON issue.fields.description}}}` as the value. That only works if the format is consistently a string, which is why the template does not ship it.

**Summary** is inserted raw with `"{{{issue.fields.summary}}}"` in both directions, so `&`, `<` and `'` arrive unescaped. Known trade-off, shared with the rest of the repository: a double quote inside a summary can still break the JSON body.

Known fidelity limitation in both directions: media nodes reference attachment ids that are local to the source instance. The attachment binary is copied to the target issue, but an inline image embedded in a description or comment will point at an id that does not exist on the target and will render as a broken media node. The file itself is still present in the target issue's attachment list. Mentions have the same problem, because `accountId` values differ per instance.

## Loop Prevention

Both instances emit the same kind of webhook, so an unguarded build loops forever: A creates, B creates, B's create webhook fires, that creates in A, and so on. The template layers five guards.

**1. Author filter, per instance.** The integration user has a different `accountId` in each instance, so each flow reads its own flow variable:

- Outgoing compares `$.issue.fields.creator.accountId` (create), `$.user.accountId` (update), `$.comment.author.accountId` (comment) and `$.payload.author.accountId` (each iterated attachment) against `{{_flow.JIRA_INTEGRATION_ACCOUNT_ID}}`, the **local** integration user.
- Incoming compares `$.user.accountId` on the root search action, so any remote event triggered by the remote integration user stops the whole flow before any write. Comment events are additionally checked on `$.comment.author.accountId`, and each iterated attachment on `$.payload.author.accountId`, against `{{_flow.REMOTE_JIRA_INTEGRATION_ACCOUNT_ID}}`, the **remote** integration user.

The per-attachment check matters: when a human later edits an issue, the flow iterates **all** attachments, including the ones the integration itself copied over from the other side. Without the attachment author check those copies would be copied back as duplicates.

**2. Link-field check on create.** Outgoing only creates a remote issue when `$.issue.fields.customfield_10153` is empty and a JQL search of the remote project on `cf[10200]` returns no issue already holding the local key. Incoming only creates a local issue when `$.issue.fields.customfield_10200` is empty and a JQL search of the local project on `cf[10153]` returned no issues.

Both create actions set the other side's link field **in the create request body**, not afterwards. That means the mirror issue's own create webhook already carries a populated link field, so this guard holds on the very first event rather than racing a later write.

**3. The write-back is itself an update event.** When a flow writes the other side's key into the link field, that fires an update on the instance that was written to:

- Outgoing action `Store Remote Issue Key on Local Issue` writes with the local iHub credential, so the resulting local update carries `$.user.accountId` equal to the local integration user and is blocked by guard 1 on `Update Remote Issue`.
- Incoming action `Store Local Issue Key on Remote Issue` writes with the remote `BASIC_AUTH` credential, so the resulting remote `jira:issue_updated` carries the remote integration user and is blocked by guard 1 on the root search action.

As a second layer that does not depend on the author being populated, both update actions also evaluate a `localLinkFieldOnlyChange` / `remoteLinkFieldOnlyChange` scripted variable. It returns `true` when every entry in `changelog.items[]` is the link field, and the update action is skipped in that case. This is a script because "every array entry equals X" cannot be expressed with the condition operators iHub supports. It repeats the link field id, so it is in the replacement list below.

**4. Issue properties for comments and attachments.** Before copying a comment or an attachment, both flows `GET` a Jira issue property on the local issue and only proceed on a `404`. After a successful copy the property is written. This makes redelivered webhooks idempotent, which matters for attachments in particular: the flows iterate the full `issue.fields.attachment[]` array, so without this guard every later update would re-upload every existing attachment.

Both directions store their properties on the **local** issue, and the ids come from two different instances whose numeric id sequences overlap. The keys are therefore namespaced: `ihub-local-<id>` for local comments and attachments sent out, `ihub-remote-<id>` for remote comments and attachments brought in.

**5. Changelog filter on incoming updates.** A remote native webhook fires a `jira:issue_updated` alongside every `comment_created`. That update has no changelog items, and `Update Local Issue` requires `$.changelog.items` to be non-empty, so a comment does not also rewrite the local summary and description.

Conflict policy is **last write wins, with no merge**. If the same field is edited on both instances at the same time, whichever webhook iHub processes last overwrites the other side. There is no field-level merge, no three-way diff and no conflict marker.

Do not install a second template that syncs the same local project anywhere else, and do not also run a Jira Automation or a second webhook on the remote instance that writes into the same project; either can reopen the loops these guards close.

## Incoming Payload

The remote instance POSTs the **standard Jira Cloud admin webhook body**; there is no custom payload contract. The incoming flow reads it directly with JSONPath and Handlebars:

| Value                       | Read with                                                     |
| --------------------------- | ------------------------------------------------------------- |
| Event type                  | `$.webhookEvent`                                              |
| Remote issue key            | `{{issue.key}}`, `$.issue.key`                                |
| Summary                     | `{{{issue.fields.summary}}}`                                  |
| Remote link field           | `$.issue.fields.customfield_10200`                            |
| Triggering user             | `$.user.accountId`                                            |
| Changelog                   | `$.changelog.items`                                           |
| Comment id                  | `{{comment.id}}`, `$.comment.id`                              |
| Comment author              | `$.comment.author.accountId`                                  |
| Attachments                 | iterator `$.issue.fields.attachment`                          |
| Description, comment body   | `remoteDescription`, `remoteCommentBody` scripts (see above)  |
| Attachment change detection | `remoteHasAttachmentChange` script                            |

`remoteHasAttachmentChange` is a script because "any changelog entry has `field` equal to `Attachment`" cannot be expressed with the supported condition operators. Jira has no dedicated attachment webhook event, so this is the only signal.

Event mapping:

| `webhookEvent`                                               | Result                             |
| ------------------------------------------------------------ | ---------------------------------- |
| `jira:issue_created`                                         | Create the issue on the other side |
| `jira:issue_updated` with field changes                      | Update summary and description     |
| `comment_created`                                            | Add the comment                    |
| `jira:issue_updated` with a changelog entry for `Attachment` | Copy attachments                   |

A representative incoming body for a comment event:

```json
{
  "webhookEvent": "comment_created",
  "issue": {
    "id": "10042",
    "key": "REMOTE-17",
    "fields": {
      "summary": "Issue summary"
    }
  },
  "comment": {
    "id": "10501",
    "author": { "accountId": "remote-user-account-id" },
    "body": "Comment text in wiki markup"
  }
}
```

**These paths are based on the documented Jira Cloud webhook shape, not on a captured delivery.** Before enabling the flows, send one event of each kind (create, update, comment, attachment) from the remote instance with the iHub flow log level raised, and confirm that `webhookEvent`, `user.accountId`, `changelog.items` and the body format match. In particular, verify whether `user` is present on `comment_created` deliveries; if it is absent, the root author check passes and the comment author check does the filtering, which is the intended behaviour.

## Credential

The manifest defines one credential: `basic-auth-remote-jira`, of type `BASIC_AUTH`. It is used by every call to the remote instance.

- **Username**: the Atlassian account email of the integration user in the remote instance. The manifest ships `integration@your-company.com` as a placeholder.
- **Password**: an Atlassian API token created by that user at `https://id.atlassian.com/manage-profile/security/api-tokens`. Paste the raw token, no prefix.

Calls to the local instance use the `ATLASSIAN_TOKEN` credential reference `ATLASSIAN_AUTH_TOKEN` and need no configuration.

The remote integration user needs Browse Projects, Create Issues, Edit Issues, Add Comments and Create Attachments permissions in the remote project, and the local integration user needs the same in the local project. Both also need to be able to edit their instance's link custom field, which must be on the create and edit screens of the synced project.

Use a dedicated account for the remote credential. Its `accountId` is the loop-guard value `REMOTE_JIRA_INTEGRATION_ACCOUNT_ID`; if a human account is used, that human's own remote edits are silently never synced.

## Remote Webhook Setup

After the template is instantiated in iHub, copy the trigger URL from the `Jira Incoming Sync` custom webhook trigger, then configure the remote instance to post to it:

1. In the remote instance go to **Settings > System > WebHooks** and choose **Create a WebHook**.
2. Set the **URL** to the iHub custom webhook trigger URL.
3. Set the JQL scope to the synced project, for example `project = REMOTE`.
4. Enable the events **Issue: created**, **Issue: updated** and **Comment: created**.
5. Leave the payload as the default Jira body. Do not enable **Exclude body**; the flow needs `issue.fields`, `changelog` and `comment`.
6. Save and enable the webhook.

Do not use a Jira Automation **Send web request** rule instead. Automation's issue data has a different shape (no `webhookEvent`, no `changelog`) and the incoming flow would not recognise it.

The local instance needs no manual webhook configuration; the Atlassian webhook trigger is wired by iHub when the flow is enabled.

## Required Customer Configuration

Create a single-line text custom field in **each** instance to hold the other instance's issue key. The template uses `customfield_10153` on the local instance and `customfield_10200` on the remote instance. These are placeholders; the ids in the customer's two instances will differ from each other and from these values.

| Value in template                                | Where                                             | Customer action                                                                                                                       |
| ------------------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `customfield_10153`                              | Both flow JSON files                              | Replace with the **local** instance link field key, for example `customfield_12345`.                                                  |
| `cf[10153]`                                      | Incoming flow JQL search                          | Replace with the numeric id of the **local** link field, for example `cf[12345]`.                                                     |
| `customfield_10200`                              | Both flow JSON files                              | Replace with the **remote** instance link field key. This is a different field from the local one.                                    |
| `cf[10200]`                                      | Outgoing flow JQL search                          | Replace with the numeric id of the **remote** link field.                                                                             |
| `REMOTE_URL`                                     | Flow variable, both flows                         | Remote Jira Cloud base URL, for example `https://acme.atlassian.net`. Default `https://your-remote-instance.atlassian.net`.           |
| `SPACE`                                          | Flow variable, incoming flow                      | Project key in the local instance that holds the synced issues. Default `LOCAL`.                                                      |
| `ISSUE_TYPE`                                     | Flow variable, incoming flow                      | Issue type name for issues created in the local instance. Default `Task`.                                                             |
| `REMOTE_PROJECT`                                 | Flow variable, outgoing flow                      | Project key in the remote instance that holds the synced issues. Default `REMOTE`.                                                    |
| `REMOTE_ISSUE_TYPE`                              | Flow variable, outgoing flow                      | Issue type name for issues created in the remote instance. Issue type names differ per instance, which is why this is separate.       |
| `JIRA_INTEGRATION_ACCOUNT_ID`                    | Flow variable, outgoing flow                      | `accountId` of the iHub integration user in the **local** instance. Default `replace-with-your-instance-of-integration-user`.         |
| `REMOTE_JIRA_INTEGRATION_ACCOUNT_ID`             | Flow variable, incoming flow                      | `accountId` of the account behind the remote credential. Default `replace-with-the-remote-integration-user`.                          |
| `integration@your-company.com`                   | `manifest.json` credential definition             | The Atlassian account email used for the remote API token, before importing or by editing the credential after import.               |

The loop-guard conditions read the account ids from these flow variables with `expectedValueSource: MUSTACHE`, so nothing inside the actions needs editing. If either is left as the placeholder, loop prevention in that direction does not work.

To find an `accountId`, call `GET /rest/api/3/myself` in each instance while authenticated as that instance's integration user.

### Places To Update The Jira Custom Fields

Local link field `customfield_10153` / `cf[10153]`:

- Outgoing: `Find Remote Issue by Local Issue Key` condition, `Store Remote Issue Key on Local Issue` body, `Link Existing Remote Issue` body, `Update Remote Issue` URL, condition and `remoteIssueKey` variable, the `localLinkFieldOnlyChange` script, `Get Local Comment ID Property` condition, `Comment Remote Issue` URL, `Get Local Attachment ID Property` condition, `Upload Attachment to Remote Issue` URL.
- Incoming: `Find Local Issue by Remote Issue Key` JQL (`cf[10153]`), `Create Local Issue` body.

Remote link field `customfield_10200` / `cf[10200]`:

- Outgoing: `Find Remote Issue by Local Issue Key` JQL (`cf[10200]`), `Create Issue in Remote Jira` body.
- Incoming: `Create Local Issue` condition, `Store Local Issue Key on Remote Issue` body, the `remoteLinkFieldOnlyChange` script.

A search-and-replace of `10153` and `10200` across both flow files covers every place.

## Security

**iHub cannot verify webhook signatures.** Jira admin webhooks can sign deliveries with a secret, but the incoming flow has no way to check that signature, so anyone who knows the iHub custom webhook URL can post a forged event. A forged `jira:issue_created` creates a local issue; a forged `comment_created` adds a comment to any linked local issue, written as the local integration user.

Treat the iHub webhook URL as a credential: do not commit it, paste it into tickets or chat, or share it. If it leaks, recreate the trigger to rotate the URL and update the remote webhook.

The only authenticity checks the flow can do are weak: it only acts on issues whose remote key it can find in `cf[10153]` (except on create) and it writes only to the `SPACE` project. Neither is a substitute for signature verification.

## Limitations

- **Status sync.** Workflows differ per instance, so a status name on one side may not exist on the other. Implementing it means calling `GET /rest/api/3/issue/{key}/transitions` on the target, matching the desired status to an available transition id, and `POST`ing to the same endpoint. A status mapping table between the two workflows is required.
- **Assignee and reporter mapping.** `accountId` values are per instance; the same person has different ids in each. This needs a user mapping table, or a lookup by email through `/rest/api/3/user/search`, which is often restricted by Atlassian privacy settings.
- **Sprints, epics and issue links.** Sprint and epic references point at board and issue ids that do not exist on the other instance. Issue links would require both linked issues to already be mirrored.
- **Priority, labels, components and other fields.** Only summary and description are synced. Adding a field means adding it to both create bodies and both update bodies; select-list values must exist with the same name on both sides.
- **Deletes and edits of comments.** Deleting an issue, comment or attachment on one side does not remove it on the other. Edited comments are not re-synced; only `comment_created` / commented events are handled.
- **Attachments added at create time.** The outgoing attachment action only runs once the local issue is linked, so attachments present on the create event are copied on the next update of that issue, not immediately.
- **Incoming formatting.** Wiki markup from the remote webhook arrives as plain text; see Body Format And Fidelity.
- **Inline media and mentions in ADF.** See Body Format And Fidelity.
- **Conflicts.** Last write wins, with no merge.
- **Unverified payload paths.** See the note under Incoming Payload. The attachment iteration relies on the current attachment's fields being visible to child actions as `{{id}}`, `{{filename}}` and `{{{content}}}`, as in the other sync templates; confirm with a test attachment in each direction.

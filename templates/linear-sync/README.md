# Jira <-> Linear Issue Sync

This template installs two iHub flows that keep Jira issues and Linear issues linked in both directions. iHub is the master of the integration: it listens to Linear webhooks and to Jira webhooks, and drives every create and update on both sides.

- `Linear Outgoing Sync`: Jira -> Linear, triggered by Jira webhooks.
- `Linear Incoming Sync`: Linear -> Jira, triggered by Linear webhooks calling an iHub custom webhook URL.

The sync stores the Linear issue UUID in a Jira custom field. Linear has no user-defined custom fields, so the link in the other direction is a Linear **attachment** pointing at the Jira browse URL. Attachments are Linear's built-in mechanism for linking external resources, and are what Linear's own GitHub and Zendesk integrations use.

Store the Linear issue `id` (a UUID such as `a1b2c3d4-...`), not the human readable `identifier` (`ENG-123`). Every Linear mutation takes the UUID. The `identifier` is still useful when reading flow logs, and both flows read it where Linear returns it.

## What The Template Syncs

From Jira to Linear:

- Looks up existing Linear attachments for the Jira browse URL with `attachmentsForURL`, so an issue that is already linked is never created twice.
- Creates a Linear issue in the configured team when a Jira issue is created and no link exists, then stores the Linear issue UUID back on the Jira issue.
- Creates a Linear attachment pointing at the Jira issue, carrying the Jira issue key and id in the attachment `metadata`.
- Optionally moves the created issue into a Linear project and an initial workflow state.
- Updates the Linear issue title and description when the Jira issue is updated.
- Adds Jira comments to the linked Linear issue.
- Adds a Linear attachment linking to each Jira attachment. The file itself is not copied, see [File Attachments](#file-attachments).
- Stores synced Linear comment and attachment ids as Jira issue properties using keys like `ihub-<linear-id>`.

From Linear to Jira:

- Searches Jira for an issue whose Linear issue id custom field matches the incoming Linear issue UUID.
- Creates a Jira issue when Linear sends an `Issue` `create` event and no matching Jira issue exists, then creates the Linear attachment that links the two.
- Updates the matching Jira issue summary and description on an `Issue` `update` event.
- Adds Linear comments to the matching Jira issue on a `Comment` `create` event.
- Stores incoming Linear comment ids as Jira issue properties using keys like `ihub-<linear-id>` to avoid duplicate processing.

## Linear API Calls

Linear has no REST API. Every call is `POST https://api.linear.app/graphql`, and the operation lives entirely in the request body, so the action name and description carry the meaning. The URL comes from the `LINEAR_API_URL` flow variable in both flows.

| Action | Flow | GraphQL operation |
| --- | --- | --- |
| Find Linear Issue by Jira URL | Outgoing | `attachmentsForURL(url:)` |
| Create Linear Issue | Outgoing | `issueCreate(input:)` |
| Link Jira Issue on Linear Issue | Outgoing and incoming | `attachmentCreate(input:)` |
| Set Linear Project on Created Issue | Outgoing | `issueUpdate(id:, input:)` |
| Set Linear Workflow State on Created Issue | Outgoing | `issueUpdate(id:, input:)` |
| Update Linear Issue | Outgoing | `issueUpdate(id:, input:)` |
| Add Jira Comment to Linear Issue | Outgoing | `commentCreate(input:)` |
| Add Jira Attachment Link to Linear Issue | Outgoing | `attachmentCreate(input:)` |

Three things about the Linear API shape the flows:

- **Values are passed as GraphQL `variables`, never interpolated into the query string.** The query text of every action is a fixed string with typed variables, and the customer data sits in the `variables` object of the request body. This keeps escaping in one place. A double quote inside a Jira summary can still break the surrounding JSON body, exactly as in the other templates in this repository.
- **Responses nest twice.** The GraphQL body is `{"data": {...}}` and iHub exposes the response body under `$.response.data`, so flow variables read paths such as `$.response.data.data.issueCreate.issue.id`.
- **Application errors are returned with HTTP 200 and an `errors` array.** Status code conditions do not catch them. Every action that branches on the success of a Linear call therefore has a data condition on `$.response.data.errors[0].message` being empty, in addition to checking that the expected id was returned.

Linear does not version its API. There is no version header and no date pinned schema, so there is no API version flow variable to configure.

## Body Format And Fidelity

Linear issue descriptions and comment bodies are **Markdown**. Jira Cloud uses ADF. iHub ships `adfToHTML` and `htmlToADF`, so neither direction converts cleanly, and the template makes a deliberate choice in each direction:

- **Jira -> Linear**: the flow does *not* send `adfToHTML` output. Raw HTML in a Markdown field renders as visible tag soup. Instead the outgoing flow uses the scripted variables `jiraDescriptionText` and `jiraCommentText`, which walk the ADF document and return plain text with paragraph and list breaks preserved, JSON escaped so it can be embedded in the request body. Bold, italics, links, tables, code blocks and inline images lose their formatting; the text survives. Panels, mentions and inline cards are reduced to their text or URL.
- **Linear -> Jira**: Markdown is passed through `htmlToADF`. The result is readable, but Markdown markers survive as literal text, so `**bold**` appears verbatim in the Jira issue.

Both flows prefix the synced body with a link back to the other system, so a reader can always jump to the original.

## File Attachments

**File contents are not copied in either direction.** Linear's `attachment` model links to external resources; it does not hold files. Real file upload in Linear is a two step, server side dance: call the `fileUpload(contentType:, filename:, size:)` mutation to get a pre-signed upload URL plus a list of headers, then `PUT` the raw bytes to that URL with exactly those headers applied, then reference the returned asset URL from Markdown. An iHub action cannot express that: the second step needs request headers that are only known at runtime from the first response, and a raw binary `PUT` to a URL returned in that response.

The degraded behaviour the template ships instead:

- Jira -> Linear: each Jira attachment produces a Linear attachment whose `url` is the Jira attachment content URL. The file stays in Jira, and a Linear user needs Jira access to open it. Like the other sync templates here, the attachment action runs on every Jira event that carries attachments, not only on the one that added a file; Linear treats the attachment `url` as an idempotent key per issue, so re-sending an already linked attachment updates it instead of creating a duplicate.
- Linear -> Jira: nothing. The incoming flow does not subscribe to Linear attachment events.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Linear issue UUID. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `11111111-1111-1111-1111-111111111111` | Outgoing `LINEAR_TEAM_ID` flow variable | Replace with the UUID of the Linear team that Jira issues are created in. Required. |
| `11111111-1111-1111-1111-111111111111` | Incoming `Find Jira Issue by Linear Issue ID` condition on `$.linearTeamId` | Replace with the same Linear team UUID. Events from other teams are dropped. |
| `22222222-2222-2222-2222-222222222222` | Incoming `Find Jira Issue by Linear Issue ID` condition on `$.linearOrganizationId` | Replace with the customer's Linear organization UUID. |
| `33333333-3333-3333-3333-333333333333` | Incoming `Find Jira Issue by Linear Issue ID` condition on `$.linearActorId` | Replace with the Linear user UUID of the integration user whose API key the template uses. This prevents circular sync. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing update, comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular sync. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `LIN` | Incoming `SPACE` flow variable | Replace with the Jira project key where Linear issues should create/find issues. |
| `Task` | Incoming `ISSUE_TYPE` flow variable | Replace if the target Jira project does not use `Task`. |
| empty | Outgoing `LINEAR_PROJECT_ID` flow variable | Optional. Set to a Linear project UUID to file created issues into that project. Left empty, the action is skipped. |
| empty | Outgoing `LINEAR_STATE_ID` flow variable | Optional. Set to a Linear workflow state UUID to give created issues a specific initial state. Left empty, the action is skipped, and the team default state applies. |

The team, organization, state and project UUIDs are all readable from the API with the integration user's key, for example:

```graphql
query {
  viewer { id name email }
  organization { id name }
  teams { nodes { id key name states { nodes { id name type } } projects { nodes { id name } } } }
}
```

The organization, team and actor values are compared inside action conditions, which match against static values, so they are edited on the conditions rather than set as flow variables.

### Places To Update The Jira Custom Field

In `linear_outgoing_sync.json`:

- `Find Linear Issue by Jira URL` condition: checks `$.issue.fields.customfield_10153` is empty before looking Linear up.
- `Store Existing Linear Issue ID on Jira Issue` and `Store Linear Issue ID on Jira Issue` bodies: write the Linear issue UUID into `customfield_10153`.
- `Update Linear Issue`, `Add Jira Comment to Linear Issue` and `Add Jira Attachment Link to Linear Issue`: read `issue.fields.customfield_10153` as the Linear issue id, and check it in their conditions.

In `linear_incoming_sync.json`:

- `Find Jira Issue by Linear Issue ID` JQL: searches `cf[10153]` for the incoming Linear issue UUID.
- `Create Jira Issue from Linear Issue` body: writes the incoming Linear issue UUID into `customfield_10153`.

## Linear Webhook Setup

After the template is instantiated in iHub, copy the trigger URL from the `Linear Incoming Sync` custom webhook trigger.

In Linear:

1. Go to `Settings` -> `API` -> `Webhooks` and create a new webhook.
2. Paste the iHub custom webhook URL as the endpoint URL.
3. Select the team the sync is configured for, or the whole workspace if the flow's team condition should do the filtering.
4. Enable the `Issues` and `Issue comments` event types. Nothing else is used by this template; enabling more only means the flow drops them.
5. Save. Linear starts delivering immediately. There is **no challenge or handshake step** on registration, unlike some other products, so there is nothing to answer and nothing to confirm in iHub.
6. Linear shows a signing secret when the webhook is created. **This template does not use it.** See the security section below.

A Linear data change event looks like this:

```json
{
  "action": "create",
  "type": "Issue",
  "createdAt": "2026-08-24T10:00:00.000Z",
  "url": "https://linear.app/acme/issue/ENG-42/login-breaks-on-safari",
  "actor": {
    "id": "44444444-4444-4444-4444-444444444444",
    "type": "user",
    "name": "Ada Lovelace"
  },
  "data": {
    "id": "a1b2c3d4-1111-2222-3333-444455556666",
    "title": "Login breaks on Safari",
    "description": "Steps to reproduce ...",
    "number": 42,
    "teamId": "11111111-1111-1111-1111-111111111111",
    "team": { "id": "11111111-1111-1111-1111-111111111111", "key": "ENG", "name": "Engineering" },
    "creatorId": "44444444-4444-4444-4444-444444444444"
  },
  "organizationId": "22222222-2222-2222-2222-222222222222",
  "webhookTimestamp": 1756029600000,
  "webhookId": "ba1c1e8f-1a2b-4c3d-9e8f-0a1b2c3d4e5f"
}
```

A `Comment` event carries the comment in `data` and the issue it belongs to in `data.issueId`:

```json
{
  "action": "create",
  "type": "Comment",
  "actor": { "id": "44444444-4444-4444-4444-444444444444", "type": "user" },
  "data": {
    "id": "c1c2c3c4-1111-2222-3333-444455556666",
    "body": "Reproduced on 17.4.",
    "issueId": "a1b2c3d4-1111-2222-3333-444455556666",
    "userId": "44444444-4444-4444-4444-444444444444",
    "issue": { "id": "a1b2c3d4-1111-2222-3333-444455556666", "identifier": "ENG-42", "title": "Login breaks on Safari" }
  },
  "organizationId": "22222222-2222-2222-2222-222222222222",
  "webhookTimestamp": 1756029600000
}
```

Event mapping used by the incoming flow:

| `type` | `action` | Result in Jira |
| --- | --- | --- |
| `Issue` | `create` | Create a Jira issue and link it back with a Linear attachment |
| `Issue` | `update` | Update the linked Jira issue summary and description |
| `Comment` | `create` | Add a Jira comment on the linked issue |

Anything else is dropped. On `update` events Linear also sends `data.updatedFrom` with the previous values of the changed properties; the template does not branch on it, it simply writes the current summary and description.

The scripted variables in the incoming flow are tolerant: they accept the payload directly or wrapped in a `payload` object, and they read the acting user from `actor.id`, `data.userId`, `data.user.id`, `data.creatorId` or `data.creator.id`, because the acting user sits in a different field per event type. Confirm the paths against real deliveries in the flow log for the event types you enable before trusting the loop guard.

## Webhook Security

Read this before enabling the incoming flow.

Linear signs every webhook delivery: a `Linear-Signature` header holding an HMAC-SHA256 of the raw request body computed with the webhook's signing secret, plus a `webhookTimestamp` field intended for replay checks.

**iHub cannot verify that signature, so this template does not.** Requests are accepted without checking `Linear-Signature`, and the signing secret Linear shows when the webhook is created is unused. There is no workaround in the flow engine; do not try to rebuild one in a scripted variable, since the raw body is not available to it.

What follows from that:

- **The only thing protecting the endpoint is that the iHub custom webhook URL is unique per flow and unguessable. Treat that URL as a credential.** Do not commit it, do not paste it into tickets, chat messages or screenshots, and do not share it beyond the people configuring the integration. If it leaks, recreate the flow trigger to rotate the URL and update the webhook endpoint in Linear.
- Because payloads are unauthenticated, anyone who learns the URL can post fabricated events and create or modify Jira issues through this flow. As a weak authenticity check, the first action of the incoming flow filters on the expected `organizationId` and the expected team UUID and drops anything that does not match. That is cheap and worth keeping regardless; it is not a substitute for signature verification.
- Replay is not detected. `webhookTimestamp` is delivered but not checked, so a captured request can be resent. The Jira issue property guard on comment ids limits duplicate comments, but a replayed `Issue` `create` for a Linear issue that has since been unlinked can produce a second Jira issue.

## Authentication

The manifest defines one Linear credential: `custom-header-linear-api-key`, of type `CUSTOM_HEADER` with a single `Authorization` header.

Create a personal API key in Linear under `Settings` -> `Security & access` -> `Personal API keys`, and paste the key as the value of the `Authorization` header when configuring the credential in iHub. The key is sent **raw, with no `Bearer` prefix**:

```
Authorization: lin_api_xxxxxxxxxxxxxxxxxxxxxxxx
```

Create the key as a dedicated integration user rather than a personal account, so the loop guard on the Linear side has a stable user UUID to filter on and so the key survives people leaving. That user needs access to the configured team.

Linear also supports OAuth2, which is the right choice for an application distributed to many workspaces, but it requires registering an OAuth application per workspace. For a template configured by one customer against one workspace, the API key is the appropriate mechanism.

## Rate Limits

Linear enforces both a request count and a query complexity budget per hour. For API key authentication the documented limits are 2,500 requests per user per hour and 3,000,000 complexity points per user per hour, with a maximum of 10,000 points for a single query. The queries in this template are small, so the request count is the limit that matters.

Exceeding a limit does **not** produce an HTTP 429 with `Retry-After`. Linear answers GraphQL requests with HTTP 400 and a `RATELIMITED` code inside the `errors` array in the response body. Every Linear facing action therefore carries a `ratelimit` block that throttles outbound calls, with both `400` and `429` in `retryStatusCodes`:

```json
"ratelimit": {
  "sleepTime": 500,
  "initialSleepTime": 5000,
  "maxRetries": 0,
  "burst": 20,
  "retryStatusCodes": [400, 408, 429, 500, 502, 503, 504]
}
```

`maxRetries` is `0`, matching the other sync templates in this repository, so the block throttles but does not retry. Raising it makes iHub retry rate limited calls, at the cost of also retrying genuinely malformed requests, which Linear answers with the same 400.

## Loop Prevention

Every write one side of the sync makes is visible to the other side as a new event. Five guards keep that from looping:

1. **Jira side actor filter.** The outgoing update, comment and attachment actions require the acting Jira account to differ from the iHub integration account (`$.payload.user.accountId` and `$.payload.author.accountId`, `$.comment.author.accountId` for comments). Confirm those paths against a real Jira event in the flow log, since the actor field differs per Jira event type.
2. **Linear side actor filter.** The first action of the incoming flow requires the Linear actor UUID to differ from the integration user's UUID, so nothing iHub writes into Linear comes back to Jira.
3. **Create only when the link is absent.** The outgoing flow first asks Linear `attachmentsForURL` for the Jira browse URL. A match routes to storing the existing Linear id on the Jira issue; only a miss creates a new Linear issue. On the incoming side, the JQL search decides between create and update the same way.
4. **Jira issue properties.** Synced comment and attachment ids are stored as Jira issue properties keyed `ihub-<linear-id>`, and the incoming comment path reads the property first and only adds the comment when it is missing (a `404`).
5. **The link write-back is itself an update.** Storing the Linear issue UUID in `customfield_10153` fires a Jira `issue_updated` webhook. Guard 1 catches it, because the update was made by the iHub integration account. If that condition's JSON path does not match your Jira events, the loop still terminates one hop later: the resulting `issueUpdate` is written by the integration user in Linear, and guard 2 drops the event it produces. Verify guard 1 in the flow log anyway, so the extra round trip does not happen on every create.

## Conflict Policy

Last write wins, with no merge. If the same issue is edited on both sides at once, whichever event iHub processes last overwrites the other side's title and description. There is no field level merge, no conflict marker and no ordering guarantee between the two flows.

## Known Limitations

Out of scope for this version of the template:

- **Workflow state mapping.** Linear workflow states are per-team UUIDs and Jira needs transitions rather than field writes, so status is not synced in either direction. `LINEAR_STATE_ID` only sets the initial state of newly created issues.
- **Assignee mapping.** Linear and Jira have separate user directories with no shared key.
- **Labels, priority, estimates, due dates.**
- **Sub-issues, issue relations, cycles, and projects** beyond filing new issues into the single configured project.
- **Multiple teams.** One flow pair handles one Linear team; the incoming flow drops events from other teams.
- **File contents.** See [File Attachments](#file-attachments).
- **Deletes.** Linear `remove` actions and Jira issue deletions are not synced.
- **Webhook signature verification and replay protection.** See [Webhook Security](#webhook-security).

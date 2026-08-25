# Microsoft Teams - Notify on Work Item Created

This template installs one iHub flow that posts a Microsoft Teams channel message whenever a work item is created in a configured Jira project.

- `Microsoft Teams - Notify on Work Item Created`: Jira -> Microsoft Teams, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from Microsoft Teams into Jira.

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in the `SPACE` flow variable.
- Posts one HTML message to the Microsoft Graph channel messages endpoint, in the team and channel configured in `TEAM_ID` and `CHANNEL_ID`.

The message contains:

- The work item key as a link back to Jira, plus the summary.
- A short list with issue type, priority, reporter and assignee.
- A closing line with the project name and the display name of the user who created the work item.

Unassigned work items render as `Unassigned` rather than an empty field or `null`, which is the normal case for a creation event. Priority falls back to `None`, since projects with priority disabled send no priority at all. Reporter and creating user have the same kind of fallback.

## Microsoft Graph API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Post channel message | `POST` | `https://graph.microsoft.com/v1.0/teams/{{_flow.TEAM_ID}}/channels/{{_flow.CHANNEL_ID}}/messages` |

The request sends `Content-Type: application/json`.

This is the Microsoft Graph API, not an Office 365 connector incoming webhook. Those connectors were retired by Microsoft and stopped working at the end of 2025, so every `MessageCard` payload and `https://outlook.office.com/webhook/...` URL still floating around is dead code. A Power Automate "Workflows" webhook URL is not used either: like any incoming webhook URL it is a secret baked into one specific channel, which has no place in a template repo.

Graph returns real HTTP status codes: `201 Created` on success, and `4xx`/`5xx` on failure with a body of `{"error": {"code": "...", "message": "..."}}`. The status code is therefore the success signal. The action additionally stores `$.response.data.id` in `teamsMessageId` (the id of the created message) and `$.response.data.error.code` in `teamsErrorCode`. Common error codes are `InvalidAuthenticationToken`, `Unauthorized`, `AccessDenied` and `ItemNotFound`.

The action carries a `ratelimit` block with `429` in `retryStatusCodes`. Graph throttles per app and per mailbox/team and answers `429` with a `Retry-After` header.

## Message Formatting

Teams channel messages are HTML, not Markdown. The body is sent as:

```json
"body": { "contentType": "html", "content": "..." }
```

Markdown renders literally, so any edit to the message must keep the HTML rules:

- Links are `<a href="https://url">link text</a>`. `[text](url)` shows up as brackets and parentheses.
- Bold is `<b>` or `<strong>`, italic is `<i>` or `<em>`.
- Teams supports only a restricted subset of HTML in channel messages. Stick to `<b>`, `<i>`, `<a>`, `<br>`, `<p>` and `<ul>`/`<li>`. `<table>`, `<style>`, `<script>` and inline CSS are not reliable.

Interpolated Jira text is written with double braces, for example `{{issue.fields.summary}}`. Handlebars escapes `&`, `<`, `>`, `"` and `'` to HTML entities, which is exactly right here: the target genuinely is HTML, so Teams decodes all five on render, and the escaped `"` also keeps the JSON request body valid. Do not switch any user-supplied value to triple braces.

## Azure AD App Registration

`POST /teams/{id}/channels/{id}/messages` **does not support application permissions.** `ChannelMessage.Send` is delegated-only, and the only application permission that can write a channel message is `Teamwork.Migrate.All`, which works solely in Teams' import/migration mode. A client-credentials credential authenticates fine and then fails every send with `Unauthorized` or `AccessDenied`. The template therefore uses the authorization-code grant with a delegated permission.

1. Go to the Azure portal -> `Microsoft Entra ID` -> `App registrations` -> `New registration`. Give it a name and register it.
2. Note the `Directory (tenant) ID` and the `Application (client) ID` from the app's `Overview` page.
3. Open `Certificates & secrets` -> `New client secret` and copy the secret value.
4. Open `API permissions` -> `Add a permission` -> `Microsoft Graph` -> `Delegated permissions`, and add `ChannelMessage.Send` and `offline_access`.
5. Choose `Grant admin consent` for the tenant.
6. Open `Authentication` -> `Add a platform` -> `Web`, and add the iHub OAuth callback URL as a redirect URI.
7. The user who authorizes the credential in iHub must be a member of the target team. Delegated permissions act as that user, so a non-member cannot post to the channel.

## How To Find The Team And Channel IDs

`TEAM_ID` and `CHANNEL_ID` are opaque IDs, not names, and there is no name-based lookup in a single call, so both must be configured.

In Teams, open the target channel, choose `...` -> `Get link to channel`, and read the link. It looks like:

```
https://teams.microsoft.com/l/channel/19%3aa1b2c3...%40thread.tacv2/General?groupId=00000000-0000-0000-0000-000000000000&tenantId=...
```

- `TEAM_ID` is the `groupId` query parameter, a GUID.
- `CHANNEL_ID` is the URL-decoded first path segment, `19:a1b2c3...@thread.tacv2`. Decode `%3a` to `:` and `%40` to `@` before pasting it in.

## OAuth2

The manifest defines one credential: `oauth2-microsoft-teams`, of type `OAUTH2` using the authorization-code grant against `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize` and `.../token`, with the scope `https://graph.microsoft.com/ChannelMessage.Send offline_access`.

`offline_access` is not optional. Without it Microsoft Entra ID issues no refresh token, and the credential stops working roughly an hour after the customer authorizes it.

Replace `{tenant}` in both URLs with the directory (tenant) ID (or the tenant domain) before importing the template, or edit the credential after import. Fill in the client ID and client secret from the app registration.

## Required Customer Configuration

Hard-coded values that must be replaced before enabling the flow:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `TEAMS` | `SPACE` flow variable | Replace with the Jira project key that should trigger Microsoft Teams notifications. |
| `00000000-0000-0000-0000-000000000000` | `TEAM_ID` flow variable | Replace with the target team's group GUID, taken from `groupId` in the channel link. |
| `19:00000000000000000000000000000000@thread.tacv2` | `CHANNEL_ID` flow variable | Replace with the target channel's thread id, URL-decoded from the channel link. |
| `{tenant}` in the OAuth2 URLs | `manifest.json` credential definition | Replace with the directory (tenant) ID or tenant domain before importing, or edit the credential after import. |

## Limitations

- The work item description is not included in the message. Converting Jira ADF to HTML is possible here, but a converted description carries `<img>` tags pointing at Jira-hosted attachment URLs that require authentication, so they render broken in Teams, and the surrounding markup uses tags outside the subset Teams supports. The link to the work item is the right affordance, and it keeps the message short enough to stay readable in a busy channel. This is a deliberate choice, not an omission.
- The message is plain HTML rather than an Adaptive Card. A card must be sent as an `attachments` entry with `contentType: "application/vnd.microsoft.card.adaptive"` whose `content` is a *stringified* JSON blob, and the body HTML must carry a matching `<attachment id="..."></attachment>` placeholder or the card renders detached from the message. That is a stringified JSON template nested inside a JSON template inside Handlebars, which is three layers of escaping for a notification that gains nothing from being a card.
- No @mentions. A mention needs a `mentions` array whose entries are keyed to matching `<at id="0">` tags in the body, and an id that does not line up fails the whole send. That is out of scope for a notification.
- One-way only. Nothing that happens in Microsoft Teams is written back to Jira.
- No threading and no message updates. Every created work item produces one new root message; later changes to the work item are not reflected in it, and the flow does not reply into the message it created.
- One project and one channel per flow. To notify several projects or channels, install the template once per pair.

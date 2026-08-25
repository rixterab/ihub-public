# Slack - Notify on Work Item Created

This template installs one iHub flow that posts a Slack message whenever a work item is created in a configured Jira project.

- `Slack - Notify on Work Item Created`: Jira -> Slack, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from Slack into Jira.

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in the `SPACE` flow variable.
- Posts one Block Kit message to `https://slack.com/api/chat.postMessage` in the channel configured in `SLACK_CHANNEL`.

The message contains:

- The work item key as a link back to Jira, plus the summary.
- A fields block with issue type, priority, reporter and assignee.
- A context line with the project name and the display name of the user who created the work item.

Unassigned work items render as `Unassigned` rather than an empty field, which is the normal case for a creation event. Priority and reporter have the same fallback.

## Slack API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Post message | `POST` | `https://slack.com/api/chat.postMessage` |

The request sends `Content-Type: application/json; charset=utf-8`. Slack rejects a JSON body sent with a form content type.

The body contains both `blocks` and `text`. `text` is required even when `blocks` is used, because it is the fallback string shown in push and desktop notifications. Without it the notification arrives blank.

`chat.postMessage` answers `HTTP 200` even when it fails. Errors arrive in the response body as `{"ok": false, "error": "channel_not_found"}`, so a status-code check reads every failure as a success. The action stores `$.response.data.ok` and `$.response.data.error` in the `slackOk` and `slackError` flow variables; use those, not the status code, when checking whether a message was delivered. Common error codes are `not_in_channel`, `channel_not_found`, `invalid_auth` and `missing_scope`.

The action carries a `ratelimit` block with `429` in `retryStatusCodes`. `chat.postMessage` is limited to roughly one message per second per channel.

## Message Formatting

Slack uses mrkdwn, not Markdown. The template follows the mrkdwn rules, and any edit to the message body must keep them:

- Bold is `*single asterisk*`, not `**double**`.
- Italic is `_underscore_`.
- Links are `<https://url|link text>`. Markdown's `[text](url)` renders literally as brackets and parens.

Interpolated Jira text is written with double braces, for example `{{issue.fields.summary}}`. That escapes `&`, `<` and `>` to `&amp;`, `&lt;` and `&gt;`, which is what Slack expects inside mrkdwn. Do not switch these to triple braces: an unescaped `&` or `<` in a summary breaks block rendering.

One caveat comes with that. The same escaping also turns `"` into `&quot;` and `'` into `&#x27;`, and Slack only decodes the three characters above, so a summary containing a straight quote or an apostrophe shows the entity in the Slack message. This is the accepted trade-off: triple braces would fix the apostrophe but let an unescaped `&` or `<` break the block, and an unescaped `"` break the JSON body.

## Slack App Setup

1. Go to `https://api.slack.com/apps` and choose `Create New App` -> `From scratch`. Give it a name and pick the workspace.
2. Open `OAuth & Permissions` and add the bot token scope `chat:write` under `Scopes` -> `Bot Token Scopes`.
3. Optionally also add `chat:write.public`. Without it the app can only post to channels it has joined.
4. Choose `Install to Workspace` and approve the app.
5. Copy the `Bot User OAuth Token` from `OAuth & Permissions`. It starts with `xoxb-`.
6. Invite the app to the target channel in Slack with `/invite @<app-name>`. `chat:write` alone fails with `not_in_channel` for a channel the app has not joined. This step can be skipped only if the app holds `chat:write.public` and the channel is public.
7. Copy the channel ID from the channel's `About` tab in Slack, at the bottom of the panel.

## Credential

The manifest defines one credential: `custom-header-slack-bot-token`, of type `CUSTOM_HEADER` with a single `Authorization` header.

Paste the full header value into the credential, including the prefix:

```
Bearer xoxb-your-bot-token
```

## Required Customer Configuration

Hard-coded values that must be replaced before enabling the flow:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `SLACK` | `SPACE` flow variable | Replace with the Jira project key that should trigger Slack notifications. |
| `C01234567` | `SLACK_CHANNEL` flow variable | Replace with the target Slack channel. A channel ID such as `C01234567` is recommended because it survives channel renames; a name such as `#jira-alerts` is also accepted. |
| Empty `Authorization` header | `custom-header-slack-bot-token` credential | Set to `Bearer xoxb-...` using the bot token from the Slack app. |

## Limitations

- The work item description is not included in the message. Jira descriptions are ADF, Slack wants mrkdwn, and there is no helper for that conversion; attempting it produces either tag soup or literal markup. The link to the work item is the right affordance, and it keeps the message short enough to stay readable in a busy channel. This is a deliberate choice, not an omission.
- The message uses a plain mrkdwn link rather than a Block Kit `button`. URL buttons dispatch an interaction payload, so if the customer's Slack app has interactivity enabled without a Request URL, clicking one shows an error to the user. A link has none of that coupling.
- One-way only. Nothing that happens in Slack is written back to Jira.
- No threading and no message updates. Every created work item produces one new message; later changes to the work item are not reflected in it, and the flow does not store the Slack `ts` needed to update or reply to a message.
- One project and one channel per flow. To notify several projects or channels, install the template once per pair.

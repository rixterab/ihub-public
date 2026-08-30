# Slack Approvals for Work Items

This template installs two iHub flows that let someone approve or reject a Jira work item from a Slack message.

- `Slack - Request Approval for Work Item`: Jira -> Slack, triggered by Jira webhooks. Posts a Block Kit message with Approve and Reject buttons.
- `Slack - Apply Approval Decision to Jira`: Slack -> Jira, triggered by Slack interactivity calling an iHub custom webhook URL. Transitions the work item, records who decided, and replaces the message so the buttons cannot be used twice.

This is the interactive counterpart to `slack-notify-work-item-created`, which only posts and never listens.

**Read [Security](#security) before enabling.** iHub cannot verify Slack's request signature, so anyone who learns the webhook URL can transition work items.

## What The Template Does

Outgoing:

- Listens for Jira `avi:jira:created:issue` events in the project in `SPACE` and of the issue type in `APPROVAL_ISSUE_TYPE`.
- Posts one message carrying the work item link, type, priority, reporter and project, plus two buttons. The Approve button has a confirmation dialog; Reject does not.
- The Jira work item key travels in each button's `value`; the decision travels in its `action_id` (`jira_approve` / `jira_reject`).

Incoming:

- Parses the interaction payload and derives `slackIssueKey`, `slackDecision`, `slackUser` and `slackResponseUrl`.
- Runs `APPROVE_TRANSITION_ID` or `REJECT_TRANSITION_ID` on the work item.
- Adds an internal Jira comment naming the Slack user who decided.
- `POST`s to Slack's `response_url` with `replace_original` so the message becomes a plain outcome line.

## Slack API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Post Approval Request to Slack | `POST` | `https://slack.com/api/chat.postMessage` |
| Replace Slack Message with Outcome | `POST` | the interaction's `response_url` |

`chat.postMessage` answers `HTTP 200` even when it fails; errors arrive as `{"ok": false, "error": "channel_not_found"}`. The action stores `$.response.data.ok` and `$.response.data.error` in `slackOk` and `slackError` — use those, not the status code. Common errors are `not_in_channel`, `channel_not_found`, `invalid_auth` and `missing_scope`.

`response_url` is pre-authorised and needs no token; the action still carries the bot-token credential, which Slack ignores on that host.

## Reading The Interaction Payload

Slack posts interactions as `application/x-www-form-urlencoded` with a single `payload` field holding a JSON string — not as a JSON body. The scripted variables therefore start by unwrapping it:

```js
let root = scope;
if (root && typeof root.payload === 'string') { root = JSON.parse(root.payload); }
else if (root && typeof root.payload === 'object') { root = root.payload; /* …and again if nested */ }
```

They then read `actions.0.value` for the key, `actions.0.action_id` for the decision, `user.name` for the actor and `response_url` for the reply target. Confirm these against a real interaction in the flow log before enabling; how iHub surfaces a form-encoded body is the one thing here worth verifying first.

**Slack requires a `200` within 3 seconds** or it shows the user a timeout error. The incoming flow's trigger is configured with a custom response of `{}` and `Content-Type: application/json` to answer immediately.

## Slack App Setup

1. Go to `https://api.slack.com/apps` -> `Create New App` -> `From scratch`.
2. Under `OAuth & Permissions` add the bot token scope `chat:write`, and optionally `chat:write.public` so the app can post without joining the channel.
3. Install to the workspace and copy the `Bot User OAuth Token` (`xoxb-…`).
4. Invite the app to the target channel with `/invite @<app-name>` unless it holds `chat:write.public` and the channel is public.
5. Under `Interactivity & Shortcuts`, turn interactivity **on** and set the **Request URL** to the custom webhook URL of the `Slack - Apply Approval Decision to Jira` flow. Without this step the buttons do nothing.
6. Copy the channel ID from the channel's `About` tab.

## Credential

One credential: `custom-header-slack-bot-token`, `CUSTOM_HEADER` with a single `Authorization` header. Paste the full value including the prefix:

```
Bearer xoxb-your-bot-token
```

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `APPROVE` | `SPACE` flow variable, outgoing | Replace with the Jira project key that requests approvals. |
| `Change` | `APPROVAL_ISSUE_TYPE` flow variable, outgoing | Replace with the issue type that requires approval. |
| `C01234567` | `SLACK_CHANNEL` flow variable, outgoing | Replace with the target channel ID. |
| empty | `APPROVE_TRANSITION_ID` flow variable, incoming | **Required.** Set to the workflow transition ID for approval. Left empty the action is skipped and the button silently does nothing. |
| empty | `REJECT_TRANSITION_ID` flow variable, incoming | **Required.** Set to the workflow transition ID for rejection. |
| Empty `Authorization` header | `custom-header-slack-bot-token` credential | Set to `Bearer xoxb-…`. |

Read the transition IDs from `GET {{baseUrl}}/rest/api/3/issue/{key}/transitions` against a work item in the state the approval starts from.

## Security

- **iHub cannot verify Slack's `X-Slack-Signature`, so this template does not.** The unguessable iHub webhook URL is the only protection. **Treat it as a credential**: anyone who learns it can transition any work item in the project by posting a crafted payload. If it leaks, recreate the flow trigger and update the Request URL in the Slack app.
- **Slack messages are not Jira permissions.** Anyone who can see the channel can click the button, regardless of whether they hold the approver role in Jira. The flow transitions as the iHub integration user, so Jira's own approval permissions are bypassed. Post approval requests to a private channel restricted to approvers, and treat this as a convenience over a controlled channel, not as an access control.
- Replay is not detected. A captured interaction can be resent, though the second attempt usually fails because the transition is no longer valid from the new status.

## Limitations

- Not JSM approvals. This runs a **workflow transition**, it does not answer a Jira Service Management approval record. A JSM approval must be answered through `/rest/servicedeskapi/request/{key}/approval/{id}`, which needs the approval ID and the approver's own credentials — neither is available here.
- One transition each way. Conditional routing by approver, amount or risk is not modelled.
- No timeout or reminder. An unanswered message stays unanswered.
- The message is not updated if the work item is transitioned in Jira instead; the buttons stay live and clicking one then fails on an invalid transition.
- One Jira project, one issue type and one channel per flow pair.

# Microsoft Teams Approvals for Work Items

This template installs one iHub flow that posts an Adaptive Card to Microsoft Teams when a Jira work item needs approval.

- `Microsoft Teams - Request Approval for Work Item`: Jira -> Teams, triggered by Jira webhooks.

The card carries the work item detail and two link buttons: one into the Jira Service Management customer portal where the approver acts, one into the Jira work item. **The decision is not made from inside Teams** — see [Why There Are No Approve And Reject Buttons](#why-there-are-no-approve-and-reject-buttons).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events in the project in `SPACE` and of the issue type in `APPROVAL_ISSUE_TYPE`.
- Posts one Adaptive Card (schema 1.4) with the work item key and summary, a `FactSet` of type, priority, reporter and project, and two `Action.OpenUrl` buttons.

## Teams API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Post Approval Card to Microsoft Teams | `POST` | `{{_flow.TEAMS_WORKFLOW_URL}}` |

The body is the `{"type": "message", "attachments": [{"contentType": "application/vnd.microsoft.card.adaptive", …}]}` envelope that the Teams **Workflows** (Power Automate) HTTP trigger expects. This is the supported successor to Office 365 connectors, which Microsoft has retired; a legacy connector `MessageCard` body will not work against a Workflows URL.

Interpolated Jira text uses double braces, matching `teams-notify-work-item-created`: Adaptive Card `TextBlock` content is escaped the same way, and the entity form is what Teams renders correctly.

## Why There Are No Approve And Reject Buttons

Three routes exist for a real in-Teams decision, and none is a good fit for a template:

- **`Action.Http` on an Actionable Message** requires registering the sending service with Microsoft's Actionable Email Developer Dashboard and having the originator approved per tenant. That is a per-customer onboarding process, not something a template can ship.
- **A Teams bot** with `Action.Execute` needs a registered bot application, an always-on messaging endpoint and per-tenant installation.
- **Power Automate approvals** move the approval out of Jira into Power Automate, which defeats the point of driving it from JSM.

The honest shape for a template is a card that carries the context and links to the place where the approval is actually recorded, with Jira's own approver permissions intact. If you want buttons in a chat client today, `slack-approval-for-work-item` implements them — and documents the security trade-off that comes with them.

## Credential

One credential: `custom-header-teams-workflow`, `CUSTOM_HEADER` with a single `Authorization` header.

A Teams Workflows HTTP trigger URL carries its own signature in the query string and normally needs **no** `Authorization` header — leave the value empty. The credential is defined for deployments that put the endpoint behind an authenticating gateway.

**Treat `TEAMS_WORKFLOW_URL` as a credential.** Anyone who learns it can post messages into the channel. Do not commit it or paste it into tickets.

## Teams Setup

1. In the target Teams channel choose `Workflows` -> `Post to a channel when a webhook request is received`.
2. Complete the wizard and copy the generated HTTP POST URL.
3. Paste it into the `TEAMS_WORKFLOW_URL` flow variable.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `APPROVE` | `SPACE` flow variable | Replace with the Jira project key that requests approvals. |
| `Change` | `APPROVAL_ISSUE_TYPE` flow variable | Replace with the issue type that requires approval. |
| `https://prod-00.westeurope.logic.azure.com:443/workflows/REPLACE_ME/…` | `TEAMS_WORKFLOW_URL` flow variable | Replace with the Workflows HTTP POST URL for the channel. |
| `https://your-site.atlassian.net/servicedesk/customer/portal/1` | `JSM_PORTAL_URL` flow variable | Replace with the customer portal URL where approvers act. |

The portal link is built as `{{_flow.JSM_PORTAL_URL}}/browse/{{issue.key}}`. Confirm that resolves on the customer's site; portal URL shapes differ between JSM configurations, and a request the approver cannot see in the portal produces a dead link.

## Limitations

- **Notification plus deep link, not an in-Teams decision.** See the section above.
- One-way. Nothing comes back from Teams into Jira, so there is no record in Jira that the card was posted or seen.
- No deduplication. A redelivered create webhook posts a second card.
- No reminder or escalation on an unanswered approval.
- The card is not updated once the approval is answered in Jira; it stays in the channel showing the original request.
- One Jira project, one issue type and one channel per flow.

# Grafana OnCall - Create Jira Incident from Alert

This template installs one iHub flow that opens a Jira incident when Grafana OnCall fires an alert group.

- `Jira Incident from Grafana OnCall Alert`: Grafana OnCall -> Jira, triggered by an OnCall outgoing webhook calling an iHub custom webhook URL.

It is the incoming counterpart to `grafana-oncall-alert-from-jira`.

## What The Template Does

- Reads the alert group ID, title, event type and permalink from the outgoing webhook payload.
- Searches Jira for an open incident already carrying that alert group ID in a custom field.
- Creates a Jira incident when none exists and the event is not a resolve.
- Adds an internal comment instead when an open incident is already linked.
- Adds a Jira remote link back to the OnCall alert group.
- Optionally transitions the incident when OnCall resolves the alert group, once a transition ID is configured.

## Grafana OnCall Setup

In Grafana OnCall go to `Outgoing Webhooks`, create a webhook of trigger type `Alert Group Created` (add `Resolved` too if you want the close path), and set the URL to the iHub custom webhook URL. Leave the data template at its default so the payload carries `alert_group`, `alert_payload`, `event` and `integration`.

The flow reads `alert_group_id`, then `alert_group.id`, then `alert_group.public_primary_key`, then `id`. `event.type` drives the routing: `resolve` maps to `resolved` and never opens a new incident; `acknowledge` and `silence` are recognised and only comment.

Confirm the field paths against a real delivery in the flow log before enabling — OnCall's payload shape varies between Grafana Cloud and self-hosted OnCall versions.

## Jira API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Find Jira Incident by Grafana OnCall Alert Key | `POST` | `{{baseUrl}}/rest/api/3/search/jql` |
| Create Jira Incident from Grafana OnCall Alert | `POST` | `{{baseUrl}}/rest/api/3/issue` |
| Comment Existing Jira Incident | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/comment` |
| Link Grafana OnCall Alert on Jira Incident | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/remotelink` |
| Transition Jira Incident on Alert Resolve | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/transitions` |

All calls use the `ATLASSIAN_TOKEN` credential; no Grafana credential is needed.

## Required Customer Configuration

Create a Jira custom field of type single-line text to hold the external alert key. The template uses `customfield_10155` / `cf[10155]`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `OPS` | `SPACE` flow variable | Replace with the Jira project key for incidents. |
| `Incident` | `ISSUE_TYPE` flow variable | Replace if the target project does not have an `Incident` type. |
| empty | `RESOLVE_TRANSITION_ID` flow variable | Optional. Set to a workflow transition ID to auto-close on resolve. |
| `customfield_10155` / `cf[10155]` | Flow JSON, three places | Replace with the customer's Jira custom field key and numeric ID. |

## Webhook Security

**iHub cannot verify an OnCall webhook signature, and this template does not try.** The iHub custom webhook URL is the only protection. **Treat it as a credential** — do not commit or share it, and recreate the flow trigger to rotate it if it leaks.

## Limitations

- Create, comment and optional close only. Acknowledgement and escalation changes only produce comments.
- No back-link into OnCall. Nothing is written from Jira into the alert group.
- OnCall payload shapes differ across versions; verify the paths in the flow log.
- One OnCall webhook and one Jira project per flow.

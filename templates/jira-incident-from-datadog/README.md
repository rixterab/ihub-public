# Datadog - Create Jira Incident from Alert

This template installs one iHub flow that opens a Jira incident when Datadog notifies a monitor alert.

- `Jira Incident from Datadog Alert`: Datadog -> Jira, triggered by a Datadog webhook calling an iHub custom webhook URL.

It is the incoming counterpart to `datadog-event-from-jira`. The two are independent; install either or both.

## What The Template Does

- Reads the alert identifier, title, transition and link from the Datadog webhook payload.
- Searches Jira for an open incident already carrying that alert ID in a custom field.
- Creates a Jira incident when none exists and the event is not a recovery.
- Adds an internal comment instead when an open incident is already linked, so a re-notify does not open a duplicate.
- Adds a Jira remote link back to the Datadog event.
- Optionally transitions the incident when Datadog sends a recovery, once a transition ID is configured.

## Datadog Webhook Setup

Datadog webhooks send a **user-defined** payload, so the template defines the contract. In Datadog go to `Integrations` -> `Webhooks`, add a webhook, paste the iHub custom webhook URL as the URL, and paste this as the payload:

```json
{
  "alert_id": "$ALERT_ID",
  "monitor_id": "$ID",
  "alert_title": "$EVENT_TITLE",
  "alert_type": "$ALERT_TYPE",
  "alert_status": "$ALERT_STATUS",
  "alert_transition": "$ALERT_TRANSITION",
  "priority": "$PRIORITY",
  "date": "$DATE",
  "org": "$ORG_NAME",
  "link": "$LINK",
  "body": "$EVENT_MSG",
  "tags": "$TAGS",
  "hostname": "$HOSTNAME"
}
```

Then reference the webhook from each monitor's notification message with `@webhook-<name>`.

The flow reads its fields tolerantly (`alert_id`, then `monitor_id`, then `alertId`, then `id`), so a payload that differs from the one above still works as long as it carries an identifier under one of those names. Confirm against a real delivery in the flow log before enabling.

`alert_transition` drives the state: `Recovered` maps to `resolved`, everything else to `firing`. A `resolved` event never opens a new incident.

## Jira API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Find Jira Incident by Datadog Alert Key | `POST` | `{{baseUrl}}/rest/api/3/search/jql` |
| Create Jira Incident from Datadog Alert | `POST` | `{{baseUrl}}/rest/api/3/issue` |
| Comment Existing Jira Incident | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/comment` |
| Link Datadog Alert on Jira Incident | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/remotelink` |
| Transition Jira Incident on Alert Resolve | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/transitions` |

All calls use the `ATLASSIAN_TOKEN` credential, so no Datadog credential is needed: the data arrives in the webhook.

## Required Customer Configuration

Create a Jira custom field of type single-line text to hold the external alert key. The template uses `customfield_10155` / `cf[10155]`, the same field the `splunk-jira-incident-alerts` template uses; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `OPS` | `SPACE` flow variable | Replace with the Jira project key for incidents. |
| `Incident` | `ISSUE_TYPE` flow variable | Replace if the target project does not have an `Incident` type. |
| empty | `RESOLVE_TRANSITION_ID` flow variable | Optional. Set to a workflow transition ID to auto-close on recovery; left empty the transition action is skipped. |
| `customfield_10155` / `cf[10155]` | Flow JSON, three places | Replace with the customer's Jira custom field key and numeric ID. |

## Webhook Security

**iHub cannot verify a Datadog webhook signature, and this template does not try.** The only thing protecting the endpoint is that the iHub custom webhook URL is unguessable. **Treat that URL as a credential**: do not commit it, paste it into tickets or chat, or share it beyond the people configuring the integration. If it leaks, recreate the flow trigger to rotate the URL and update the webhook in Datadog. Anyone who learns it can create Jira incidents.

## Limitations

- Create, comment and optional close only. Datadog alert acknowledgement, priority changes and tag changes are not synced.
- No back-link into Datadog. Nothing is written from Jira into the Datadog event.
- The transition action does not check the current status, so a workflow where the configured transition is not available from the incident's current status fails on that one call. The incident itself is unaffected.
- One Datadog org and one Jira project per flow.

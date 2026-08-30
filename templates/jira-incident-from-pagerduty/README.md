# PagerDuty - Create Jira Incident from Incident

This template installs one iHub flow that opens a Jira incident when PagerDuty triggers an incident.

- `Jira Incident from PagerDuty Incident`: PagerDuty -> Jira, triggered by a PagerDuty v3 webhook calling an iHub custom webhook URL.

It is the incoming counterpart to `pagerduty-incident-from-jira`. **Install only one direction per service**, or a PagerDuty incident created from Jira will create a Jira incident which pages again. See [Loop Risk](#loop-risk).

## What The Template Does

- Reads the PagerDuty incident ID, title, event type and `html_url` from the v3 webhook envelope.
- Searches Jira for an open incident already carrying that incident ID in a custom field.
- Creates a Jira incident when none exists and the event is not a resolve.
- Adds an internal comment instead when an open incident is already linked.
- Adds a Jira remote link back to the PagerDuty incident.
- Optionally transitions the incident when PagerDuty resolves, once a transition ID is configured.

## PagerDuty Setup

In PagerDuty go to `Integrations` -> `Generic Webhooks (v3)`, add a webhook subscription scoped to the service or account, and set the delivery URL to the iHub custom webhook URL. Subscribe to `incident.triggered`, `incident.resolved` and optionally `incident.acknowledged`.

The v3 envelope is `{"event": {"id":…, "event_type": "incident.triggered", "data": {"id":…, "title":…, "html_url":…, "status":…, "urgency":…, "service": {…}, "priority": {…}}}}`. The flow reads `event.data.id` first and falls back to `data.id`, `incident.id` and `id`, so v2 payloads and hand-rolled tests also work.

`event.event_type` drives the routing: anything containing `resolve` maps to `resolved` and never opens a new Jira incident.

## Jira API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Find Jira Incident by PagerDuty Alert Key | `POST` | `{{baseUrl}}/rest/api/3/search/jql` |
| Create Jira Incident from PagerDuty Alert | `POST` | `{{baseUrl}}/rest/api/3/issue` |
| Comment Existing Jira Incident | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/comment` |
| Link PagerDuty Alert on Jira Incident | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/remotelink` |
| Transition Jira Incident on Alert Resolve | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/transitions` |

All calls use the `ATLASSIAN_TOKEN` credential; no PagerDuty credential is needed.

## Loop Risk

Running this flow and `pagerduty-incident-from-jira` against the same PagerDuty service and the same Jira project creates a cycle: Jira work item -> PagerDuty incident -> Jira incident. The two templates have no shared loop guard.

If you want both directions, separate them: use different Jira projects, or scope the PagerDuty webhook subscription to a service that the outgoing template does not write to. Verify in the flow log before enabling both.

## Required Customer Configuration

Create a Jira custom field of type single-line text to hold the external alert key. The template uses `customfield_10155` / `cf[10155]`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `OPS` | `SPACE` flow variable | Replace with the Jira project key for incidents. |
| `Incident` | `ISSUE_TYPE` flow variable | Replace if the target project does not have an `Incident` type. |
| empty | `RESOLVE_TRANSITION_ID` flow variable | Optional. Set to a workflow transition ID to auto-close on resolve. |
| `customfield_10155` / `cf[10155]` | Flow JSON, three places | Replace with the customer's Jira custom field key and numeric ID. |

## Webhook Security

PagerDuty signs v3 deliveries with an `X-PagerDuty-Signature` header (HMAC-SHA256 of the raw body). **iHub cannot verify that signature, so this template does not.** The iHub custom webhook URL is the only protection. **Treat it as a credential** — do not commit or share it, and recreate the flow trigger to rotate it if it leaks. Replay is not detected either.

## Limitations

- Create, comment and optional close only. Acknowledgement, reassignment and priority changes only produce comments.
- No back-link into PagerDuty. Nothing is written from Jira into the incident.
- One PagerDuty subscription and one Jira project per flow.

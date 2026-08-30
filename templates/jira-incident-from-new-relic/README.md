# New Relic - Create Jira Incident from Alert

This template installs one iHub flow that opens a Jira incident when a New Relic workflow notifies an issue.

- `Jira Incident from New Relic Alert`: New Relic -> Jira, triggered by a New Relic workflow webhook calling an iHub custom webhook URL.

It is the incoming counterpart to `new-relic-event-from-jira`.

## What The Template Does

- Reads the New Relic issue ID, title, state and issue URL from the workflow payload.
- Searches Jira for an open incident already carrying that issue ID in a custom field.
- Creates a Jira incident when none exists and the issue is not closed.
- Adds an internal comment instead when an open incident is already linked.
- Adds a Jira remote link back to the New Relic issue.
- Optionally transitions the incident when New Relic closes the issue, once a transition ID is configured.

## New Relic Workflow Setup

In New Relic go to `Alerts` -> `Workflows`, create a workflow, filter it to the policies that should raise Jira incidents, and add a **Webhook** destination pointing at the iHub custom webhook URL.

The default New Relic payload already works. Its relevant fields are `id`, `issueUrl`, `title` (an array), `state` (`ACTIVATED` / `CLOSED`), `priority`, `alertPolicyNames`, `alertConditionNames`, `impactedEntities`, `totalIncidents` and `createdAt`. The flow reads each of these tolerantly and falls back to snake_case variants, so a custom payload template also works provided it keeps an identifier under `id`, `issueId` or `incidentId`.

`state` drives the routing: anything containing `CLOSED` maps to `resolved` and never opens a new incident.

New Relic sends a test notification when you save the destination. Expect one test incident in Jira, or send the test before pointing the workflow at a real policy.

## Jira API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Find Jira Incident by New Relic Alert Key | `POST` | `{{baseUrl}}/rest/api/3/search/jql` |
| Create Jira Incident from New Relic Alert | `POST` | `{{baseUrl}}/rest/api/3/issue` |
| Comment Existing Jira Incident | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/comment` |
| Link New Relic Alert on Jira Incident | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/remotelink` |
| Transition Jira Incident on Alert Resolve | `POST` | `{{baseUrl}}/rest/api/3/issue/{key}/transitions` |

All calls use the `ATLASSIAN_TOKEN` credential; no New Relic credential is needed.

## Required Customer Configuration

Create a Jira custom field of type single-line text to hold the external alert key. The template uses `customfield_10155` / `cf[10155]`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `OPS` | `SPACE` flow variable | Replace with the Jira project key for incidents. |
| `Incident` | `ISSUE_TYPE` flow variable | Replace if the target project does not have an `Incident` type. |
| empty | `RESOLVE_TRANSITION_ID` flow variable | Optional. Set to a workflow transition ID to auto-close when the New Relic issue closes. |
| `customfield_10155` / `cf[10155]` | Flow JSON, three places | Replace with the customer's Jira custom field key and numeric ID. |

## Webhook Security

**iHub cannot verify a New Relic webhook signature, and this template does not try.** The iHub custom webhook URL is the only protection. **Treat it as a credential** — do not commit or share it, and recreate the flow trigger to rotate it if it leaks. New Relic does support a custom header on the destination; adding a shared secret there and checking it is not something this flow does today.

## Limitations

- Create, comment and optional close only. Acknowledgement and priority changes are not synced.
- New Relic `title` is an array. The flow reads `title.0` first and falls back to the whole value, so a multi-condition issue shows only the first title.
- No back-link into New Relic. Nothing is written from Jira into the issue.
- One workflow and one Jira project per flow.

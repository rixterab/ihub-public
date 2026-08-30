# Datadog - Post Event from Jira

This template installs one iHub flow that posts a Datadog event when a work item is created in a configured Jira project.

- `Datadog - Post Event from Jira`: Jira -> Datadog, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from Datadog into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in `SPACE`.
- Posts one event with `alert_type: info` and `source_type_name: jira`, tagged so it can be correlated on dashboards.

## Datadog API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Post Datadog Event | `POST` | `{{_flow.DATADOG_API_URL}}/api/v1/events` |

The `DATADOG_API_URL` flow variable must match your Datadog site, otherwise the API key is rejected: `https://api.datadoghq.com` (US1), `https://api.datadoghq.eu` (EU1), `https://api.us3.datadoghq.com`, `https://api.us5.datadoghq.com`, `https://api.ddog-gov.com`.

## Tags

Datadog tags are one string each, in `key:value` form. The event carries four:

| Tag | Source |
| --- | --- |
| `source:<DATADOG_TAG_SOURCE>` | Flow variable, default `ihub` |
| `team:<DATADOG_TAG_TEAM>` | Flow variable, default `platform` |
| `jira_project:<project key>` | Jira event |
| `jira_issue:<issue key>` | Jira event |

A flow variable holding `a,b` produces the single literal tag `a,b`, not two tags. To add more tags, add further entries to the `tags` array in the action body rather than packing them into one variable.

## Credential

The manifest defines one credential: `custom-header-datadog-api-key`, of type `CUSTOM_HEADER` with a single `DD-API-KEY` header.

Create an API key in Datadog under `Organization Settings` -> `API Keys` and paste the raw key as the header value. No prefix. The events endpoint needs an **API key**, not an application key.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `DATADOG` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `https://api.datadoghq.com` | `DATADOG_API_URL` flow variable | Replace with the API URL for your Datadog site. |
| `ihub` | `DATADOG_TAG_SOURCE` flow variable | Replace with the value for the `source:` tag. |
| `platform` | `DATADOG_TAG_TEAM` flow variable | Replace with the value for the `team:` tag. |
| Empty `DD-API-KEY` header | `custom-header-datadog-api-key` credential | Set to a Datadog API key. |

## Limitations

- One-way only. Datadog monitors and alerts are not turned into Jira work items. Pair this with a flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a Datadog webhook if you want ingestion into Jira.
- Create only. Updates, transitions and closures do not produce further events, so a Datadog dashboard built on this sees openings but never closures.
- Events only. This does not create a Datadog incident or Case, and does not page anyone.
- The Jira description is not sent. Jira descriptions are ADF and the event `text` field is plain text with limited Datadog markdown; the link back to Jira is the right affordance.
- `/api/v1/events` is the classic events intake. Datadog is moving toward the v2 events API; migrate the URL and body if your organisation has standardised on it.

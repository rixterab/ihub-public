# Datadog - Post Event from Jira

Post a Datadog event when a Jira work item is created in the configured project.

The template installs one flow:

- `Datadog - Post Event from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `DATADOG` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `https://api.datadoghq.com` | `DATADOG_API_URL` flow variable | Replace with the correct Datadog site, for example `https://api.datadoghq.eu`. |
| `jira,ihub` | `DATADOG_TAGS` flow variable | Replace with tags used by your observability dashboards. |
| `Datadog API Key` | `manifest.json` credential | Add an API key authorized to submit events. |

This is a lightweight event template. For alert ingestion into Jira, pair it with a webhook-triggered Jira creation flow.

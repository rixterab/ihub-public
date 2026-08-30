# New Relic - Post Event from Jira

This template installs one iHub flow that posts a custom event into New Relic when a work item is created in a configured Jira project.

- `New Relic - Post Event from Jira`: Jira -> New Relic, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from New Relic into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in `SPACE`.
- Posts one custom event of type `JiraWorkItemEvent` carrying the issue key, project key, summary, priority and browse URL.

## New Relic API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Post New Relic Event | `POST` | `{{_flow.NEW_RELIC_INSERT_URL}}/v1/accounts/{{_flow.NEW_RELIC_ACCOUNT_ID}}/events` |

Use the insert URL for your region: `https://insights-collector.newrelic.com` (US) or `https://insights-collector.eu01.nr-data.net` (EU).

The Event API answers `200` with `{"success": true}`. It accepts the payload and validates asynchronously, so a rejected attribute is **not** visible in the flow log; check `NrIntegrationError` in the account if events do not appear.

Attribute rules worth knowing before editing the body: `eventType` must be alphanumeric plus `:` and `_`, attribute values must be strings or numbers (no nested objects or arrays), and there is a limit of 255 attributes per event.

## Querying The Events

```sql
SELECT count(*) FROM JiraWorkItemEvent FACET jiraProjectKey SINCE 1 week ago
```

Custom events are retained for a limited period on most account tiers; do not treat this as an audit log.

## Credential

The manifest defines one credential: `custom-header-new-relic-insert-key`, of type `CUSTOM_HEADER` with a single `Api-Key` header.

Create a **license key** or an **Insights insert key** in New Relic under `API keys` and paste it raw as the header value. No prefix. `X-Insert-Key` is the legacy header name for this endpoint; `Api-Key` is the current one and is what this template sends.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `NEWRELIC` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `1234567` | `NEW_RELIC_ACCOUNT_ID` flow variable | Replace with the New Relic account ID. |
| `https://insights-collector.newrelic.com` | `NEW_RELIC_INSERT_URL` flow variable | Replace with the EU endpoint if your account is in the EU region. |
| Empty `Api-Key` header | `custom-header-new-relic-insert-key` credential | Set to a New Relic license or insert key. |

## Limitations

- One-way only. New Relic alerts are not turned into Jira work items. Pair this with a flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a New Relic workflow if you want ingestion into Jira.
- Create only. Updates, transitions and closures produce no further events, so NRQL built on this sees openings but never closures.
- Custom events only. This does not open a New Relic incident and does not page anyone.
- The Jira description is not sent. It is ADF, and event attributes must be flat strings.
- Failures are asynchronous and invisible to the flow log. See the API section above.

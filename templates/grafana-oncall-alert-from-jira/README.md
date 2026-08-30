# Grafana OnCall - Create Alert from Jira

This template installs one iHub flow that posts an alert to a Grafana OnCall inbound integration when a work item of a configured priority is created in a configured Jira project.

- `Grafana OnCall - Create Alert from Jira`: Jira -> Grafana OnCall, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from Grafana into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in `SPACE`.
- Ignores every work item whose priority is not the one configured in `GRAFANA_TRIGGER_PRIORITY`.
- Posts one alert to the OnCall inbound integration configured in `GRAFANA_ONCALL_INTEGRATION_ID`.

## Grafana OnCall API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Grafana OnCall Alert | `POST` | `{{_flow.GRAFANA_ONCALL_URL}}/integrations/v1/{{_flow.GRAFANA_ONCALL_INTEGRATION_ID}}/` |

The trailing slash is required. This is the formatted-webhook inbound integration endpoint, not the OnCall REST API, so the payload shape is **whatever the integration's alert template expects**. The template sends a reasonable default:

| Field | Value |
| --- | --- |
| `title` | Jira key and summary |
| `message` | Link back to the Jira work item |
| `state` | `alerting` |
| `severity` | `GRAFANA_SEVERITY` flow variable |
| `jira_priority` | The Jira priority name |
| `jira_issue_key`, `jira_project_key` | Grouping keys |

`severity` and the Jira priority are separate fields on purpose. Jira priority names (`High`, `Highest`) are not Grafana severities, so sending the Jira value into `severity` produces a severity Grafana does not recognise. Set `GRAFANA_SEVERITY` to a value your OnCall route template understands, and route on `jira_priority` if you want per-priority behaviour.

Adjust the grouping, title and message templates in the OnCall integration to match these field names.

## Credential

The manifest defines one credential: `custom-header-grafana-oncall-token`, of type `CUSTOM_HEADER` with a single `Authorization` header.

Grafana OnCall inbound integration URLs are unguessable and are commonly called without authentication, in which case leave the header value empty. If your deployment sits behind an authenticating proxy or you use the Grafana Cloud API, paste the full header value including its prefix, for example `Bearer glsa_...`.

**Treat the integration URL as a credential.** Anyone who learns it can raise alerts and page your on-call. Do not commit it or paste it into tickets or chat.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `GRAFANA` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `High` | `GRAFANA_TRIGGER_PRIORITY` flow variable | Replace with the Jira priority name that should alert. |
| `https://oncall-prod-us-central-0.grafana.net` | `GRAFANA_ONCALL_URL` flow variable | Replace with your Grafana OnCall API URL. |
| `ABC123` | `GRAFANA_ONCALL_INTEGRATION_ID` flow variable | Replace with the inbound integration ID from the integration's URL. |
| `critical` | `GRAFANA_SEVERITY` flow variable | Replace with a severity your OnCall route template understands. |

## Limitations

- One-way only. Resolving the OnCall alert group does not change the Jira work item, and resolving the Jira work item does not resolve the alert group.
- No deduplication. A redelivered create webhook posts a second alert. OnCall groups by its own alert-group rules, not by anything this flow sends.
- The inbound integration endpoint returns `200` for a payload the alert template cannot render, so a malformed payload is not visible in the flow log. Verify against a real alert in the OnCall UI.
- One Jira project, one priority and one integration per flow.

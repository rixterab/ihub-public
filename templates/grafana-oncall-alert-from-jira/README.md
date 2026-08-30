# Grafana OnCall - Create Alert from Jira

Create a Grafana OnCall alert when a high-priority Jira work item is created.

The template installs one flow:

- `Grafana OnCall - Create Alert from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `GRAFANA` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `High` | Action condition | Replace with the Jira priority that should create an alert. |
| `https://oncall-prod-us-central-0.grafana.net` | `GRAFANA_ONCALL_URL` flow variable | Replace with your Grafana OnCall API URL. |
| `ABC123` | `GRAFANA_ONCALL_INTEGRATION_ID` flow variable | Replace with the inbound integration ID. |
| `Grafana OnCall Token` | `manifest.json` credential | Paste the authorization header value expected by your Grafana OnCall API. |

This template uses an inbound integration endpoint. Adjust the payload to match the route template configured in Grafana OnCall.

# Alertmanager - Create Silence from Jira

Create a Prometheus Alertmanager silence when a Jira change work item is created.

The template installs one flow:

- `Alertmanager - Create Silence from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `CHANGE` | `SPACE` flow variable | Replace with the Jira project key for change work. |
| `Change` | Action condition | Replace with the Jira issue type that should create silences. |
| `https://alertmanager.example.com` | `ALERTMANAGER_URL` flow variable | Replace with the Alertmanager base URL. |
| `service` / `example-service` | Flow variables | Replace with the label matcher used by your alert rules. |
| `2027-01-01T00:00:00Z` / `2027-01-01T02:00:00Z` | Flow variables | Replace manually or map from Jira change-window custom fields. |

Most production use cases should map silence start and end times from Jira custom fields and add stricter approval/status conditions before enabling.

# New Relic - Post Event from Jira

Post a Jira work item created event into New Relic as a custom event.

The template installs one flow:

- `New Relic - Post Event from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `NEWRELIC` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `1234567` | `NEW_RELIC_ACCOUNT_ID` flow variable | Replace with the New Relic account ID. |
| `https://insights-collector.newrelic.com` | `NEW_RELIC_INSERT_URL` flow variable | Replace for EU or other New Relic regions. |
| `New Relic Insert Key` | `manifest.json` credential | Add a New Relic Events API insert key. |

Use NRQL on `JiraWorkItemEvent` to build dashboards or alert policies around Jira activity.

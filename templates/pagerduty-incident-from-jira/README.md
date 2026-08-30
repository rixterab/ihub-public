# PagerDuty - Create Incident from Jira

Create a PagerDuty incident when a high-priority Jira work item is created.

The template installs one flow:

- `PagerDuty - Create Incident from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `PAGERDUTY` | `SPACE` flow variable | Replace with the Jira project key that should create PagerDuty incidents. |
| `Critical` | Action condition | Replace with the Jira priority that should page. |
| `P123456` | `PAGERDUTY_SERVICE_ID` flow variable | Replace with the PagerDuty service ID. |
| `integration@company.com` | `PAGERDUTY_FROM_EMAIL` flow variable | Replace with an email for the PagerDuty `From` header. |
| `PagerDuty API Token` | `manifest.json` credential | Paste the full `Token token=...` authorization value. |

Add escalation policy, urgency, or assignment fields if your PagerDuty process requires them.

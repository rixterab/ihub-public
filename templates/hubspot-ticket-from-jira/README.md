# HubSpot - Create Ticket from Jira

Create a HubSpot ticket when a Jira work item is created in the configured project.

The template installs one flow:

- `HubSpot - Create Ticket from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `HUBSPOT` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `0` | `HUBSPOT_PIPELINE_ID` flow variable | Replace with the target HubSpot ticket pipeline ID. |
| `1` | `HUBSPOT_STAGE_ID` flow variable | Replace with the target ticket stage ID. |
| `HubSpot Private App Token` | `manifest.json` credential | Paste a `Bearer ...` private app token with ticket write access. |

Add association actions if the ticket should be linked to HubSpot companies, contacts, or deals.

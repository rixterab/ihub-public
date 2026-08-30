# Dynamics 365 - Create Case from Jira

Create a Microsoft Dynamics 365 case when a Jira work item is created in the configured project.

The template installs one flow:

- `Dynamics 365 - Create Case from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `DYNAMICS` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `https://your-org.crm.dynamics.com` | `DYNAMICS_365_URL` flow variable | Replace with your Dynamics environment URL. |
| `Dynamics 365 Authorization` | `manifest.json` credential | Paste a `Bearer ...` token or adapt to your OAuth credential pattern. |

The template creates an `incident` record. Adjust field names if your Dynamics environment uses custom columns or required fields.

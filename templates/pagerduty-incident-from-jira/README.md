# PagerDuty - Create Incident from Jira

This template installs one iHub flow that creates a PagerDuty incident when a work item of a configured priority is created in a configured Jira project.

- `PagerDuty - Create Incident from Jira`: Jira -> PagerDuty, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from PagerDuty into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in `SPACE`.
- Ignores every work item whose priority is not the one configured in `PAGERDUTY_TRIGGER_PRIORITY`.
- Creates one incident on the PagerDuty service configured in `PAGERDUTY_SERVICE_ID`, with `urgency: high`.

## PagerDuty API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Create PagerDuty Incident | `POST` | `https://api.pagerduty.com/incidents` |

The `Accept: application/vnd.pagerduty+json;version=2` header selects the REST API v2 payload shape, and the `From` header is **required** by this endpoint: PagerDuty attributes the incident to that user and rejects the request with `400` if the address does not belong to a user in the account. `PAGERDUTY_FROM_EMAIL` must therefore be a real PagerDuty user, not a generic alias.

The action stores `$.response.data.incident.id` in `pagerDutyIncidentId`. It is not written back to Jira; see [Limitations](#limitations).

## Credential

The manifest defines one credential: `custom-header-pagerduty-token`, of type `CUSTOM_HEADER` with a single `Authorization` header.

Create a REST API key in PagerDuty under `Integrations` -> `API Access Keys`. PagerDuty does not use a `Bearer` prefix; paste the full header value including the `Token token=` prefix:

```
Authorization: Token token=y_NbAkKc66ryYTWUXYEu
```

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `PAGERDUTY` | `SPACE` flow variable | Replace with the Jira project key that should page. |
| `Critical` | `PAGERDUTY_TRIGGER_PRIORITY` flow variable | Replace with the Jira priority name that should page. It must match the priority name exactly. |
| `P123456` | `PAGERDUTY_SERVICE_ID` flow variable | Replace with the PagerDuty service ID. |
| `integration@company.com` | `PAGERDUTY_FROM_EMAIL` flow variable | Replace with the login email of a real PagerDuty user. |
| Empty `Authorization` header | `custom-header-pagerduty-token` credential | Set to `Token token=...`. |

## Limitations

- One-way only. Acknowledging or resolving the PagerDuty incident does not change the Jira work item, and resolving the Jira work item does not resolve the incident.
- The incident ID is captured in a flow variable but not stored on the Jira work item, so there is no link to follow later. Add a Jira write-back action if you plan to resolve the incident from Jira, following the pattern in `gitlab-issue-from-jira`.
- No deduplication. A redelivered create webhook pages a second time. PagerDuty's `incident_key` is the right fix if that matters; set it to the Jira issue key.
- `urgency` is fixed at `high`, and escalation policy, assignment and conference bridge are not set. The service's own escalation policy applies.
- One Jira project, one priority and one PagerDuty service per flow.

# Alertmanager - Create Silence from Jira

This template installs one iHub flow that creates a Prometheus Alertmanager silence when a change work item is created in a configured Jira project.

- `Alertmanager - Create Silence from Jira`: Jira -> Alertmanager, triggered by Jira webhooks.

**Read [Limitations](#limitations) before enabling this flow.** A silence suppresses alerting, so a misconfigured window is a monitoring outage.

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in `SPACE`.
- Ignores every work item whose issue type is not the one configured in `CHANGE_ISSUE_TYPE`.
- Creates one silence matching a single label, with a comment linking back to the Jira work item.
- Writes the returned silence ID back to a Jira custom field, so the silence can be found and expired later.

## Alertmanager API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Alertmanager Silence | `POST` | `{{_flow.ALERTMANAGER_URL}}/api/v2/silences` |
| Store Alertmanager Silence ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` |

The create action stores `$.response.data.silenceID` in `alertmanagerSilenceId`. Expiring a silence is `DELETE /api/v2/silence/{silenceID}`, which is why the ID is written back.

## Silence Window

`startsAt` and `endsAt` must be RFC3339 timestamps. The body prefers Jira change-window custom fields and falls back to flow variables:

```
"startsAt": "{{#if issue.fields.customfield_10160}}{{issue.fields.customfield_10160}}{{else}}{{_flow.SILENCE_STARTS_AT}}{{/if}}"
"endsAt":   "{{#if issue.fields.customfield_10161}}{{issue.fields.customfield_10161}}{{else}}{{_flow.SILENCE_ENDS_AT}}{{/if}}"
```

Map `customfield_10160` and `customfield_10161` to the planned start and end date-time fields on your change request type. Jira date-time fields serialise as `2026-08-30T14:00:00.000+0200`, which Alertmanager accepts.

The `SILENCE_STARTS_AT` / `SILENCE_ENDS_AT` fallbacks are placeholders, not a sensible default. Leaving them in place silences the matcher for a fixed two-hour window in 2030 on **every** matching change, which is almost certainly not what you want.

## Credential

The manifest defines one credential: `custom-header-alertmanager-auth`, of type `CUSTOM_HEADER` with a single `Authorization` header.

Alertmanager has no built-in authentication. In practice it sits behind a reverse proxy or an ingress that does, so paste the full header value that proxy expects, including its prefix, for example `Basic ...` or `Bearer ...`. If the endpoint is unauthenticated, leave the value empty and make sure iHub can reach it.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the silence ID. The template uses `customfield_10153`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `CHANGE` | `SPACE` flow variable | Replace with the Jira project key for change work. |
| `Change` | `CHANGE_ISSUE_TYPE` flow variable | Replace with the Jira issue type that represents a change window. |
| `https://alertmanager.example.com` | `ALERTMANAGER_URL` flow variable | Replace with the Alertmanager base URL. |
| `service` | `ALERTMANAGER_MATCHER_NAME` flow variable | Replace with the alert label your rules match on. |
| `example-service` | `ALERTMANAGER_MATCHER_VALUE` flow variable | Replace with the label value to silence. |
| `2030-01-01T...` | `SILENCE_STARTS_AT` / `SILENCE_ENDS_AT` flow variables | Fallbacks only. Map the real window from Jira custom fields. |
| `customfield_10160` / `customfield_10161` | Flow JSON body | Replace with the change-window start and end field keys. |
| `customfield_10153` | Flow JSON, one place | Replace with the customer's Jira custom field key. |

## Limitations

- **No approval check.** The flow silences on work item *creation*, before anyone has approved the change. Add a condition on approval status, or move the trigger to a status transition, before enabling this in production.
- **Nothing expires the silence early.** Cancelling or closing the Jira change does not delete the silence; it runs to `endsAt`. The stored silence ID is what a second flow would use to call `DELETE /api/v2/silence/{id}`.
- **One matcher, exact match.** `isRegex` is `false` and only one label is matched. A change touching several services needs several matchers, added to the `matchers` array in the body.
- No deduplication. A redelivered create webhook creates a second silence; Alertmanager does not merge them.
- One Jira project and one issue type per flow.

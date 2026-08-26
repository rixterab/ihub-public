# Splunk <-> Jira Incident Alerts

This template installs two iHub flows for Jira incidents and Splunk alerts.

- `Jira Incident to Splunk Alert`: Jira -> Splunk, triggered by Jira webhooks. It creates a Splunk saved-search alert when a Jira incident is created.
- `Splunk Unacknowledged Alert to Jira Incident`: Splunk -> Jira, triggered by a Splunk webhook. It creates a Jira incident when Splunk reports an alert that is still unacknowledged after the configured threshold.

The flows share one Jira custom field as the link key. The template currently uses `customfield_10155` / `cf[10155]` to store a Splunk saved-search name, alert id, search name, or SID.

## What The Template Does

From Jira to Splunk:

- Listens for Jira `avi:jira:created:issue` events in the configured project.
- Runs only for the configured Jira incident issue type.
- Skips Jira incidents created by the iHub integration user to avoid loops.
- Creates a Splunk saved-search alert through the Splunk management API.
- Stores the created Splunk alert key back on the Jira incident.
- Adds an internal Jira comment with the Splunk saved-search alert name.

From Splunk to Jira:

- Accepts Splunk webhook payloads from a saved-search alert or a separate "unacknowledged alert monitor" search.
- Normalizes common Splunk webhook fields such as `search_name`, `sid`, `results_link`, `owner`, `app`, and `result`.
- Accepts optional acknowledgement fields such as `acknowledged`, `ack_status`, `status`, `state`, `ageMinutes`, `age_minutes`, `first_triggered_at`, `trigger_time`, and `result._time`.
- Searches Jira for an open incident whose Splunk alert key field matches the alert.
- Creates a Jira incident when no open linked incident exists.
- Comments on the existing Jira incident when the same unacknowledged alert is received again.
- Adds a Jira remote link to Splunk results when the webhook includes `results_link`.

## Splunk API Calls

The Jira -> Splunk flow uses the Splunk management API. The default endpoint is:

| Action | Method | Endpoint |
| --- | --- | --- |
| Create saved-search alert | `POST` | `{{_flow.SPLUNK_URL}}/servicesNS/{{_flow.SPLUNK_OWNER}}/{{_flow.SPLUNK_APP}}/saved/searches?output_mode=json` |

The request body is `application/x-www-form-urlencoded` and includes saved-search alert settings such as `name`, `search`, `cron_schedule`, `dispatch.earliest_time`, `dispatch.latest_time`, `alert_type`, `alert_comparator`, `alert_threshold`, `alert.track`, and `alert.severity`.

The manifest defines a custom `Authorization` header credential named `Splunk Authorization Header`. Paste the full header value expected by the Splunk management API, for example `Bearer <token>` or `Basic <base64-username-password>`.

## Acknowledgement Logic

Use time as the main rule and status as a guard:

- Time rule: create the Jira incident only after the Splunk alert has been unacknowledged for `ACK_TIMEOUT_MINUTES`.
- Status guard: do not create a Jira incident when the payload says the alert is acknowledged, closed, resolved, or cleared.

The flow enforces this when Splunk sends `ageMinutes`, `age_minutes`, `first_triggered_at`, `trigger_time`, or `result._time`. If none of those fields exists, the flow assumes the Splunk saved search already filtered the results to "open/unacknowledged and older than the threshold."

Recommended Splunk pattern: create a separate Splunk saved-search alert that finds stale unacknowledged alerts or notable events, and attach the iHub custom webhook URL from `Splunk Unacknowledged Alert to Jira Incident`. That keeps the acknowledgement rule in Splunk, where the alert status data lives.

## Splunk Webhook Payload

Splunk's standard webhook alert action sends JSON with `sid`, `results_link`, `search_name`, `owner`, `app`, and the first result row under `result`.

Recommended payload when you control the unacknowledged monitor search output:

```json
{
  "search_name": "Database Error Rate",
  "sid": "scheduler_admin_search_W2_at_14232356_132",
  "results_link": "https://splunk.example.com/app/search/@go?sid=scheduler_admin_search_W2_at_14232356_132",
  "owner": "admin",
  "app": "search",
  "result": {
    "title": "Database Error Rate",
    "status": "unacknowledged",
    "age_minutes": "21",
    "severity": "critical",
    "host": "db-01",
    "message": "Error rate above threshold"
  }
}
```

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Splunk alert key. The template currently uses `customfield_10155` / `cf[10155]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `OPS` | Both `SPACE` flow variables | Replace with the Jira project key for incidents. |
| `Incident` | `INCIDENT_ISSUE_TYPE` and `ISSUE_TYPE` flow variables | Replace if the Jira project uses another issue type name. |
| `customfield_10155` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10155]` | Splunk -> Jira JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Jira -> Splunk flow variable | Replace with the Jira account ID used by iHub/the integration user. |
| `https://your-splunk-host:8089` | Jira -> Splunk `SPLUNK_URL` flow variable | Replace with the Splunk management API URL. Splunk Cloud commonly uses `https://<deployment-name>.splunkcloud.com:8089`. |
| `nobody` / `search` | Jira -> Splunk `SPLUNK_OWNER` and `SPLUNK_APP` flow variables | Replace with the Splunk namespace that should own the saved-search alert. |
| `search index=main (error OR failure) \| head 100` | Jira -> Splunk `SPLUNK_DEFAULT_SEARCH` flow variable | Replace with the real SPL query. Tokens `${JIRA_ISSUE_KEY}` and `${JIRA_SUMMARY}` are substituted before the alert is created. |
| `*/5 * * * *`, `-15m`, `now` | Jira -> Splunk scheduling variables | Adjust the saved-search cadence and dispatch window. |
| `15` | Splunk -> Jira `ACK_TIMEOUT_MINUTES` flow variable | Adjust the unacknowledged threshold. |
| Empty `SPLUNK_CREATED_ALERT_WEBHOOK_URL` | Jira -> Splunk flow variable | Optional. Set only if alerts created from Jira should also call a webhook action. |
| Empty `Authorization` header | `custom-header-splunk-authorization` credential | Set to the full Splunk REST authorization header value. |

## Splunk Setup Notes

After importing the template, copy the trigger URL from the `Splunk Unacknowledged Alert to Jira Incident` custom webhook trigger. Attach it to the Splunk saved-search alert that represents stale, unacknowledged alerts.

Splunk Enterprise 9.0 and newer can require the webhook URL to be added to the webhook allow list before a webhook alert action may send to it. Configure that in Splunk before enabling the alert action.

Splunk Cloud REST API access uses the management port. Some deployments require Splunk Support to open REST API access on port 8089.

References:

- Splunk saved-search REST endpoint: https://help.splunk.com/en/splunk-enterprise/leverage-rest-apis/rest-api-reference/10.4/search-endpoints/search-endpoint-descriptions
- Splunk webhook alert action payload: https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.0/configure-alert-actions/use-a-webhook-alert-action
- Splunk webhook allow list: https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.4/manage-alert-and-alert-action-permissions/configure-webhook-allow-list

## Limitations

- This is not a full two-way sync. It creates and links records, but it does not close Splunk alerts when Jira incidents resolve or close Jira incidents when Splunk alerts clear.
- The Jira -> Splunk flow creates saved-search alerts from one configured SPL query. Customers must tune `SPLUNK_DEFAULT_SEARCH` for their environment.
- Standard Splunk webhook payloads do not always include acknowledgement state or age. For reliable escalation, make the Splunk monitor search return only alerts that are still unacknowledged and older than the SLA threshold.
- Splunk Enterprise Security, ITSI, Observability Cloud, and On-Call use different data models for alert or notable-event acknowledgement. Keep the iHub webhook payload contract above, and adapt the Splunk SPL/source system to emit those fields.

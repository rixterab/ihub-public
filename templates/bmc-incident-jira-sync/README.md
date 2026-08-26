# Sync BMC Incident <-> Jira Issue

This template installs two iHub flows that keep Jira issues and BMC Helix ITSM incidents linked in both directions.

- `BMC Incident Incoming Sync`: BMC -> Jira, triggered by a BMC webhook, filter, workflow, or middleware call to an iHub custom webhook URL.
- `BMC Incident Outgoing Sync`: Jira -> BMC, triggered by Jira webhooks.

The sync stores the BMC incident number in a Jira custom field. That field is the link key used by both flows to find the matching record.

## What The Template Syncs

From BMC to Jira:

- Searches Jira for an issue whose BMC incident number custom field matches the incoming incident.
- Creates a Jira issue when BMC sends an `incident.created` event and no matching Jira issue exists.
- Updates the matching Jira issue summary and description on an `incident.updated` event.
- Adds BMC work log comments to the matching Jira issue.
- Downloads BMC work log attachments and uploads them to the matching Jira issue.
- Stores incoming BMC comment and attachment IDs as Jira issue properties using keys like `ihub-<bmc-id>`.

From Jira to BMC:

- Creates a BMC incident when a Jira issue is created and the Jira issue does not already have a BMC incident number.
- Stores the created BMC incident number back on the Jira issue.
- Updates the BMC incident description fields when the Jira issue is updated.
- Adds Jira comments to the linked BMC incident as `HPD:WorkLog` entries.
- Uploads Jira attachments to the linked BMC incident as work log attachments.
- Stores synced Jira comment and BMC work log IDs as Jira issue properties to avoid duplicate processing.

## BMC API Calls

The flows use the BMC Helix ITSM / Remedy AR System REST API. The default forms follow the BMC incident interface forms:

| Action | Method | Endpoint |
| --- | --- | --- |
| Create incident | `POST` | `{{_flow.BMC_URL}}/api/arsys/v1/entry/{{_flow.BMC_INCIDENT_CREATE_FORM}}?fields=values(Incident%20Number)` |
| Update incident | `PUT` | `{{_flow.BMC_URL}}/api/arsys/v1/entry/{{_flow.BMC_INCIDENT_UPDATE_FORM}}/{{issue.fields.customfield_10153}}|{{issue.fields.customfield_10153}}` |
| Add work log | `POST` | `{{_flow.BMC_URL}}/api/arsys/v1/mergeEntry/{{_flow.BMC_WORK_LOG_FORM}}?fields=values(Request%20ID)` |
| Download attachment | `GET` | `{{_flow.BMC_URL}}/api/arsys/v1/entry/{{_flow.BMC_WORK_LOG_FORM}}/{{bmcAttachmentEntryId}}/attach/{{bmcAttachmentFieldName}}` |

The manifest defines a custom `Authorization` header credential named `BMC Authorization Header`. For AR-JWT authentication, paste the full value such as `AR-JWT <token>`. For Helix SSO/OAuth-backed environments, paste the full bearer value required by the tenant.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the BMC incident number. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://your-company-restapi.onbmc.com` | Incoming and outgoing `BMC_URL` flow variable | Replace with the customer's BMC Helix ITSM / AR System REST API base URL. |
| `HPD:IncidentInterface_Create` | Outgoing `BMC_INCIDENT_CREATE_FORM` flow variable | Replace if the customer uses a customized incident create interface form. |
| `HPD:IncidentInterface` | Outgoing `BMC_INCIDENT_UPDATE_FORM` flow variable | Replace if the customer uses a customized incident update interface form. |
| `HPD:WorkLog` | Incoming and outgoing `BMC_WORK_LOG_FORM` flow variable | Replace if work information is handled by a customized form. |
| `Attachment` | Incoming and outgoing `BMC_WORK_LOG_ATTACHMENT_FIELD` flow variable | Replace with the attachment field name used on the customer's work log form. |
| `Allen`, `Allbrook`, `Calbro Services` | Outgoing requester variables | Replace with valid BMC requester values for incidents created from Jira. |
| `3-Moderate/Limited`, `3-Medium`, `Assigned`, `Direct Input`, `User Service Restoration` | Outgoing BMC incident defaults | Replace with values allowed by the customer's BMC configuration. |
| `BMC` | Incoming `SPACE` flow variable | Replace with the Jira project key where BMC incidents should create/find issues. |
| `Task` | Incoming `ISSUE_TYPE` flow variable | Replace if the target Jira project does not use `Task`. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular comment and attachment sync. |

## BMC Webhook Payload

After the template is instantiated in iHub, copy the trigger URL from the `BMC Incident Incoming Sync` custom webhook trigger. Configure BMC workflow, a filter, an escalation, or middleware to POST JSON payloads to that URL.

Recommended payload for an incident create event:

```json
{
  "eventType": "incident.created",
  "incidentNumber": "INC000000000701",
  "summary": "Incident short description",
  "description": "Incident details"
}
```

Recommended payload for an incident update event:

```json
{
  "eventType": "incident.updated",
  "incidentNumber": "INC000000000701",
  "summary": "Updated incident short description",
  "description": "Updated incident details"
}
```

Recommended payload for a work log comment event:

```json
{
  "eventType": "incident.commented",
  "incidentNumber": "INC000000000701",
  "workLogId": "WLG000000000602",
  "commentBody": "Work log text"
}
```

Recommended payload for an attachment event:

```json
{
  "eventType": "incident.attachment.created",
  "incidentNumber": "INC000000000701",
  "attachmentId": "WLG000000000602-Attachment",
  "attachmentEntryId": "WLG000000000602",
  "attachmentFieldName": "Attachment",
  "fileName": "example.pdf"
}
```

The incoming flow also accepts common BMC-style key variants such as `Incident Number`, `values.Incident Number`, `Description`, `Detailed_Decription`, `Request ID`, `workLogId`, `work_info_id`, `attachmentEntryId`, and nested `incident`, `comment`, `workLog`, and `attachment` objects.

## Attachment Notes

BMC attachment handling is form-specific. The template assumes incident attachments are exposed from work log entries with the AR System attachment endpoint `/entry/{formName}/{entryId}/attach/{fieldName}`. If the customer's webhook sends a direct download URL instead, update the `Download BMC Attachment` action URL to use that payload field.

Jira-to-BMC attachment upload uses multipart form data with an `entry` part and an `attach-{{_flow.BMC_WORK_LOG_ATTACHMENT_FIELD}}` part. Verify the attachment field name against the customer's `HPD:WorkLog` form before enabling attachment sync.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. Incoming BMC events check Jira issue properties before creating comments or attachments.

On the BMC side, restrict webhook/filter logic so it does not fire for changes made by the integration user. Without that condition, updates iHub writes into BMC can be sent straight back to Jira.

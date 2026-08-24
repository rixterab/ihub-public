# Jira <-> Salesforce Case Sync

This template installs two iHub flows that keep Jira issues and Salesforce Cases linked in both directions.

- `Salesforce Outgoing Sync`: Jira -> Salesforce, triggered by Jira webhooks.
- `Salesforce Incoming Sync`: Salesforce -> Jira, triggered by a Salesforce Flow posting to an iHub custom webhook URL.

The sync stores the Salesforce Case ID in a Jira custom field. That field is the link key used by both flows to find the matching record.

## What The Template Syncs

From Jira to Salesforce:

- Creates a Salesforce Case when a Jira issue is created and the Jira issue does not already have a Salesforce Case ID.
- Stores the created Salesforce Case ID back on the Jira issue.
- Updates the Salesforce Case `Description` when the Jira issue is updated.
- Adds Jira comments to the linked Salesforce Case using `CaseComment`.
- Uploads Jira attachments to the linked Salesforce Case as Salesforce Files using `ContentVersion`.
- Stores synced Salesforce comment and attachment IDs as Jira issue properties using keys like `ihub-<salesforce-id>` to avoid duplicate processing.

From Salesforce to Jira:

- Searches Jira for an issue whose Salesforce Case ID custom field matches the incoming Case ID.
- Creates a Jira issue when Salesforce sends a `case.created` event and no matching Jira issue exists.
- Adds Salesforce Case comments to the matching Jira issue.
- Downloads Salesforce Files by `ContentVersionId` and uploads them to the matching Jira issue.
- Stores incoming Salesforce comment and attachment IDs as Jira issue properties using keys like `ihub-<salesforce-id>`.

## Salesforce API Calls

The outgoing flow uses OAuth2 and Salesforce REST sObject endpoints:

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Case | `POST` | `{{_flow.SF_URL}}/services/data/{{_flow.SF_API_VERSION}}/sobjects/Case` |
| Comment Case | `POST` | `{{_flow.SF_URL}}/services/data/{{_flow.SF_API_VERSION}}/sobjects/CaseComment` |
| Update Case Description | `PATCH` | `{{_flow.SF_URL}}/services/data/{{_flow.SF_API_VERSION}}/sobjects/Case/{{issue.fields.customfield_10153}}` |
| Upload Case File | `POST` | `{{_flow.SF_URL}}/services/data/{{_flow.SF_API_VERSION}}/sobjects/ContentVersion` |

The file upload uses `ContentVersion` with `FirstPublishLocationId` set to the Salesforce Case ID, so Salesforce links the uploaded File to the Case.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Salesforce Case ID. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://your-domain.my.salesforce.com` | Incoming and outgoing `SF_URL` flow variable | Replace with the customer's Salesforce My Domain or instance URL. |
| `v67.0` | Incoming and outgoing `SF_API_VERSION` flow variable | Replace if the Salesforce org must use another REST API version. |
| `Jira` | Outgoing `CASE_ORIGIN` flow variable | Must match an allowed Salesforce Case `Origin` picklist value. |
| `New` | Outgoing `CASE_STATUS` flow variable | Must match an allowed Salesforce Case `Status` picklist value. |
| `SF` | Incoming `SPACE` flow variable | Replace with the Jira project key where Salesforce-created cases should create/find issues. |
| `Task` | Incoming `ISSUE_TYPE` flow variable | Replace if the target Jira project does not use `Task`. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular comment and attachment sync. |

## Salesforce Flow Payload

After the template is instantiated in iHub, copy the trigger URL from the `Salesforce Incoming Sync` custom webhook trigger. Configure Salesforce Flow to send JSON payloads to that URL.

Recommended payload for a Case create event:

```json
{
  "eventType": "case.created",
  "caseId": "500XXXXXXXXXXXX",
  "subject": "Case subject",
  "description": "Case description"
}
```

Recommended payload for a Case comment event:

```json
{
  "eventType": "case.commented",
  "caseId": "500XXXXXXXXXXXX",
  "commentId": "00aXXXXXXXXXXXX",
  "commentBody": "Comment text"
}
```

Recommended payload for a Case file event:

```json
{
  "eventType": "case.attachment.created",
  "caseId": "500XXXXXXXXXXXX",
  "contentVersionId": "068XXXXXXXXXXXX",
  "fileName": "example.pdf"
}
```

The incoming flow also accepts common Salesforce-style key variants such as `CaseId`, `recordId`, `Case.Id`, `CommentBody`, `ContentVersionId`, and nested `payload` objects, but the payloads above are the intended contract.

## OAuth2

The manifest defines one Salesforce OAuth2 credential: `oauth2-salesforce`. It requests the `api refresh_token` scopes and is used by every Salesforce REST call in both flows.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. Incoming Salesforce events also check Jira issue properties before creating comments or attachments. This prevents events created by one side of the sync from being immediately sent back to the source side.

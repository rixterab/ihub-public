# Jira -> Scrive -> Jira Signature

This template installs two iHub flows that send a Jira agreement issue to Scrive for signature, then update the same Jira issue from Scrive webhook status changes.

- `Scrive Outgoing Signature`: Jira -> Scrive, triggered when a Jira issue moves to `Send for Signature`.
- `Scrive Incoming Status`: Scrive -> Jira, triggered by Scrive webhook events.

The Jira-side behavior matches the existing Jira signing workflow. The provider-side API host and paths are flow variables because signing providers expose different create-from-template, webhook, and signed-PDF download shapes across products and environments.

## Flow Behavior

From Jira to Scrive:

- Watches Jira issue update webhooks.
- Detects a status change to `Send for Signature`.
- Calls the configured Scrive create/signature-request API path.
- Maps signer email and signer name from Jira custom fields.
- Stores the returned Scrive signature request id in a Jira custom field.
- Writes initial Jira signature status `Sent`.
- Adds a Jira trace comment with the created signature request id.

From Scrive to Jira:

- Receives Scrive webhook events.
- Reads a signature request id and status from common webhook payload locations.
- Finds the Jira issue whose signature request id custom field matches the webhook.
- Updates a Jira signature status custom field:
  - provider `sent` / `pending` -> Jira `Sent`
  - provider `delivered` / `viewed` / `opened` -> Jira `Delivered`
  - provider `completed` / `signed` -> Jira `Signed`
- On completion, downloads the signed PDF from the configured Scrive download path and attaches it to Jira.
- Adds the comment `Agreement signed by customer`.
- Transitions the Jira issue to `Completed`.
- Stores an issue property named `ihub-scrive-<signatureRequestId>-completed` so webhook retries do not duplicate the PDF, comment, or transition.

## Required Jira Fields

Create or identify these Jira custom fields before enabling the flows. The field IDs below are placeholders and must be replaced for each Jira site.

| Template placeholder | Meaning | Expected type |
| --- | --- | --- |
| `customfield_10201` | Customer signer email | Single-line text or email field |
| `customfield_10202` | Customer signer name | Single-line text |
| `customfield_10203` / `cf[10203]` | Scrive signature request id | Single-line text |
| `customfield_10204` | Signature status (`Sent`, `Delivered`, `Signed`) | Single-line text |

Replace the field IDs in both JSON files.

## Flow Variables

| Variable | Flow | Description |
| --- | --- | --- |
| `SPACE` | Both | Jira project key containing agreement issues. |
| `SCRIVE_BASE_URL` | Both | Scrive API host for the target account or environment. |
| `SCRIVE_CREATE_SIGNATURE_PATH` | Outgoing | API path that creates and sends a signature request from a template. |
| `SCRIVE_TEMPLATE_ID` | Outgoing | Scrive template ID used for the agreement. |
| `SCRIVE_SIGNER_ROLE_NAME` | Outgoing | Signer role name used by the template, if supported by Scrive. |
| `SCRIVE_WEBHOOK_URL` | Outgoing | Custom webhook trigger URL from the incoming flow. |
| `SCRIVE_DOWNLOAD_SIGNED_PDF_PATH_PREFIX` | Incoming | API path prefix before the signature request id for downloading the signed PDF. |
| `SCRIVE_DOWNLOAD_SIGNED_PDF_PATH_SUFFIX` | Incoming | API path suffix after the signature request id for downloading the signed PDF. |
| `COMPLETED_TRANSITION_ID` | Incoming | Jira workflow transition ID that moves the issue to `Completed`. |

## Scrive API Configuration

The template uses a custom `Authorization` header credential named `Scrive Authorization Header`. After import, paste the full header value required by your Scrive account, for example a `Bearer ...`, `API-Key ...`, or other provider-specific authorization value.

Before enabling the flows, verify these provider-specific details against your Scrive tenant and API version:

- The create/signature-request endpoint and request body.
- The JSONPath used to store the created request id. The template defaults to `$.response.data.id`.
- The webhook payload paths for request id and status. The incoming flow handles common fields such as `data.id`, `signature_request.id`, `agreementId`, `documentId`, `status`, `state`, and `event_type`.
- The signed PDF download path prefix and suffix.

## iHub Setup

After importing the template:

1. Create the `Scrive Incoming Status` flow first and copy its custom webhook trigger URL.
2. Paste that URL into the outgoing flow variable `SCRIVE_WEBHOOK_URL`.
3. Set the Jira project key, Scrive API host, template ID, create path, and download path variables.
4. Replace all Jira custom field placeholders with the field IDs from the target Jira site.
5. Adjust the create request body and response id JSONPath if the Scrive endpoint requires a different shape.
6. Set `COMPLETED_TRANSITION_ID` to the transition ID for the Jira workflow path to `Completed`.
7. Enable both flows.

## Notes

- The outgoing flow only creates a signature request when the Jira issue has no stored request id. Clearing the field and moving the issue back to `Send for Signature` will send a new request.
- If the provider uses workflow statuses that should move Jira issues directly, replace the signature-status field update action with workflow transition actions.
- The completion path is idempotent after the completion property is stored. If a run fails after attaching the PDF but before storing the property, a provider webhook retry can repeat the remaining steps.

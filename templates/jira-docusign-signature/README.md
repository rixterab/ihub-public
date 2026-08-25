# Jira -> DocuSign -> Jira Signature

This template installs two iHub flows that send a Jira agreement issue to DocuSign for signature, then update the same Jira issue from DocuSign webhook status changes.

- `DocuSign Outgoing Signature`: Jira -> DocuSign, triggered when a Jira issue moves to `Send for Signature`.
- `DocuSign Incoming Status`: DocuSign -> Jira, triggered by DocuSign Connect webhook events.

The intended workflow is:

```text
Jira issue moves to "Send for Signature"
-> iHub creates a DocuSign envelope
-> iHub stores envelopeId on the Jira issue
-> DocuSign webhook reports status changes
-> iHub updates Jira signature status: Sent -> Delivered -> Signed
-> when completed, iHub attaches the signed PDF to Jira
-> iHub adds "Agreement signed by customer"
-> iHub transitions the Jira issue to Completed
```

## Flow Behavior

From Jira to DocuSign:

- Watches Jira issue update webhooks.
- Detects a status change to `Send for Signature`.
- Creates and sends a DocuSign envelope from a configured DocuSign server template.
- Maps signer email and signer name from Jira custom fields.
- Stores the returned DocuSign `envelopeId` in a Jira custom field.
- Writes initial Jira signature status `Sent`.
- Adds a Jira trace comment with the created `envelopeId`.

From DocuSign to Jira:

- Receives DocuSign Connect JSON webhook events.
- Reads the `envelopeId` and envelope status from the webhook payload.
- Finds the Jira issue whose envelopeId custom field matches the webhook.
- Updates a Jira signature status custom field:
  - DocuSign `sent` -> Jira `Sent`
  - DocuSign `delivered` -> Jira `Delivered`
  - DocuSign `completed` -> Jira `Signed`
- On `completed`, downloads the combined signed PDF from DocuSign and attaches it to Jira.
- Adds the comment `Agreement signed by customer`.
- Transitions the Jira issue to `Completed`.
- Stores an issue property named `ihub-docusign-<envelopeId>-completed` so DocuSign webhook retries do not duplicate the PDF, comment, or transition.

## Jira API Calls

| Purpose | Method | Endpoint |
| --- | --- | --- |
| Store envelope/status fields | `PUT` | `{{baseUrl}}/rest/api/3/issue/{issueKey}` |
| Find issue by envelopeId | `POST` | `{{baseUrl}}/rest/api/3/search/jql` |
| Attach signed PDF | `POST` | `{{baseUrl}}/rest/api/3/issue/{issueKey}/attachments` |
| Add signed comment | `POST` | `{{baseUrl}}/rest/api/3/issue/{issueKey}/comment` |
| Transition to Completed | `POST` | `{{baseUrl}}/rest/api/3/issue/{issueKey}/transitions` |
| Store completion idempotency property | `PUT` | `{{baseUrl}}/rest/api/3/issue/{issueKey}/properties/ihub-docusign-{envelopeId}-completed` |

Jira attachment uploads use `multipart/form-data` and the required `X-Atlassian-Token: no-check` header.

## DocuSign API Calls

| Purpose | Method | Endpoint |
| --- | --- | --- |
| Create and send envelope | `POST` | `{{DOCUSIGN_BASE_URL}}/restapi/v2.1/accounts/{accountId}/envelopes` |
| Download signed combined PDF | `GET` | `{{DOCUSIGN_BASE_URL}}/restapi/v2.1/accounts/{accountId}/envelopes/{envelopeId}/documents/combined?certificate=true` |

The outgoing flow includes an envelope-level `eventNotification` so DocuSign posts `sent`, `delivered`, and `completed` status changes to the incoming iHub custom webhook URL. If your DocuSign account already uses an account-level Connect configuration, you can remove `eventNotification` from the create-envelope body and point the account-level Connect configuration to the incoming flow instead.

## Required Jira Fields

Create or identify these Jira custom fields before enabling the flows. The field IDs below are placeholders and must be replaced for each Jira site.

| Template placeholder | Meaning | Expected type |
| --- | --- | --- |
| `customfield_10201` | Customer signer email | Single-line text or email field |
| `customfield_10202` | Customer signer name | Single-line text |
| `customfield_10203` / `cf[10203]` | DocuSign envelopeId | Single-line text |
| `customfield_10204` | Signature status (`Sent`, `Delivered`, `Signed`) | Single-line text |

Replace the field IDs in both JSON files:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `customfield_10201` | Outgoing flow create-envelope body and conditions | Replace with the Jira signer email field ID. |
| `customfield_10202` | Outgoing flow create-envelope body and conditions | Replace with the Jira signer name field ID. |
| `customfield_10203` | Both flows | Replace with the Jira envelopeId field ID. |
| `cf[10203]` | Incoming flow JQL | Replace with the numeric ID of the same envelopeId field. |
| `customfield_10204` | Both flows | Replace with the Jira signature status field ID. |

## Flow Variables

| Variable | Flow | Description |
| --- | --- | --- |
| `SPACE` | Both | Jira project key containing agreement issues. |
| `DOCUSIGN_BASE_URL` | Both | DocuSign API host. Use `https://demo.docusign.net` for developer accounts; replace with the production account base URI when going live. |
| `DOCUSIGN_ACCOUNT_ID` | Both | DocuSign API Account ID GUID. |
| `DOCUSIGN_TEMPLATE_ID` | Outgoing | DocuSign server template ID used for the agreement. |
| `DOCUSIGN_SIGNER_ROLE_NAME` | Outgoing | Role name in the DocuSign template, for example `Customer`. |
| `DOCUSIGN_CONNECT_URL` | Outgoing | Custom webhook trigger URL from the incoming flow. |
| `COMPLETED_TRANSITION_ID` | Incoming | Jira workflow transition ID that moves the issue to `Completed`. |

## DocuSign Setup

1. Create a DocuSign server template for the agreement.
2. Add a signer role whose role name matches `DOCUSIGN_SIGNER_ROLE_NAME`.
3. Optionally add text tabs labeled `JiraIssueKey` and `JiraSummary`; the outgoing flow populates these if they exist.
4. Create a DocuSign OAuth app/integration key for iHub and authorize it with the `signature` scope. The manifest uses DocuSign developer OAuth URLs by default.
5. For production, change the manifest OAuth URLs from `account-d.docusign.com` to `account.docusign.com` before publishing or importing in a production tenant.

## iHub Setup

After importing the template:

1. Create the `DocuSign Incoming Status` flow first and copy its custom webhook trigger URL.
2. Paste that URL into the outgoing flow variable `DOCUSIGN_CONNECT_URL`.
3. Set the Jira project key, DocuSign account ID, base URL, template ID, and signer role name.
4. Replace all Jira custom field placeholders with the field IDs from the target Jira site.
5. Set `COMPLETED_TRANSITION_ID` to the transition ID for the Jira workflow path to `Completed`.
6. Enable both flows.

## Notes

- The outgoing flow only creates an envelope when the Jira issue has no stored envelopeId. Clearing the field and moving the issue back to `Send for Signature` will send a new envelope.
- The incoming flow updates a Jira custom field for `Sent`, `Delivered`, and `Signed`. If those should be Jira workflow statuses instead, replace the status-field update action with workflow transition actions for each target status.
- The completion path is idempotent after the completion property is stored. If a run fails after attaching the PDF but before storing the property, a DocuSign retry can repeat the remaining steps.

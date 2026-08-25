# SAP S/4HANA

Procurement, purchase order, business partner, and exception-management templates for SAP S/4HANA and Jira.

The template installs four flows:

- `SAP S/4HANA - Create Purchase Requisition`: Jira -> SAP. When a JSM purchase request reaches the configured approved status, iHub creates a purchase requisition in SAP S/4HANA and writes the SAP PR number back to Jira.
- `SAP S/4HANA - Get Purchase Order`: Jira -> SAP -> Jira. Reads the SAP PO number from Jira, fetches the current PO details from SAP, and updates Jira fields/comments.
- `SAP S/4HANA - Sync Business Partner`: SAP -> Jira Assets. Scheduled import of SAP Business Partner records into Jira Assets.
- `SAP S/4HANA - Create Jira Issue from SAP Exception`: SAP -> Jira. Receives SAP or middleware exception events and creates/updates Jira issues so Jira becomes the exception-management layer.

## Required Customer Configuration

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://my-s4hana.example.com` | `S4_URL` variables | Replace with the customer's SAP S/4HANA base URL. |
| `API_PURCHASEREQ_PROCESS_SRV` | `PR_SERVICE` | Confirm the enabled purchase requisition OData service/path. |
| `API_PURCHASEORDER_PROCESS_SRV` | `PO_SERVICE` | Confirm the enabled purchase order OData service/path. |
| `API_BUSINESS_PARTNER` | `BUSINESS_PARTNER_SERVICE` | Confirm the enabled Business Partner OData service/path. |
| `PROC`, `SAP` | `SPACE` variables | Replace with the target Jira project keys. |
| `Approved` | `APPROVED_STATUS` variables | Replace with the exact Jira approval-complete status. |
| `Task` | `ISSUE_TYPE` | Replace with the Jira issue type for SAP exceptions. |
| `customfield_10154` | Create PR flow | Jira field that stores the SAP purchase requisition number. Also update the matching condition. |
| `customfield_10155` | Get PO flow | Jira field that stores the SAP purchase order number. |
| `customfield_10156`, `customfield_10157`, `customfield_10158` | Get PO flow | Jira fields for SAP PO status, supplier, and net amount. |
| `customfield_10160` to `customfield_10168` | Create PR flow scripts | Jira request fields for item text, material, quantity, unit, plant, purchasing group, delivery date, cost center, and G/L account. |
| `customfield_10153`, `customfield_10154`, `cf[10153]` | Exception flow | Jira fields/JQL used for SAP object ID and object type. Replace with customer field IDs. |
| `00000000-0000-0000-0000-000000000000` | Business Partner `importSourceId` | Replace with the Jira Assets external import source ID. |
| `SAP_EXCEPTION_CALLBACK_URL` | Exception flow | Optional callback URL. The acknowledgement action is disabled until this is configured. |

## SAP Exception Webhook Payload

After importing the template, copy the trigger URL from `SAP S/4HANA - Create Jira Issue from SAP Exception` and configure SAP Event Mesh, SAP Integration Suite, Cloud ALM, or another middleware layer to post exception events to iHub.

Recommended payload:

```json
{
  "sapObjectId": "5000001234",
  "sapObjectType": "Sales Order",
  "process": "Order Fulfillment",
  "severity": "High",
  "summary": "Sales order 5000001234 failed credit release",
  "description": "Credit status could not be updated because the business partner is blocked."
}
```

The flow also accepts variants such as `objectId`, `object_id`, `businessObject.id`, `event.objectId`, `detail.objectId`, `message`, and `errorMessage`.

## Credentials

- `basic-auth-sap-s4hana`: SAP communication user used for OData reads/writes and optional exception callbacks.
- `token-sap-s4-business-partner-asset-import-token`: Jira Assets external import token.

## Notes

- SAP OData write APIs commonly require a CSRF token. The purchase requisition flow fetches one before the create call.
- SAP field names, services, and entity sets can differ by S/4HANA edition and communication arrangement. Treat the shipped service paths as working defaults to verify, not customer-agnostic constants.
- The Business Partner import is intentionally single-page (`PAGE_SIZE`). Add pagination if the customer's Business Partner volume exceeds the configured limit.

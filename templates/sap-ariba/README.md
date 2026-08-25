# SAP Ariba

Purchase request, supplier onboarding, and purchase status templates for SAP Ariba and Jira Service Management.

The template installs three flows:

- `SAP Ariba - Create Purchase Request`: Jira -> Ariba. When a Jira purchase request reaches the configured approved status, iHub creates an Ariba purchase request and writes the Ariba request ID back to Jira.
- `SAP Ariba - Supplier Onboarding`: Jira -> Ariba. When a supplier request is approved in Jira, iHub creates an Ariba supplier onboarding request and writes the Ariba supplier request ID back to Jira.
- `SAP Ariba - Purchase Status to Jira`: Ariba -> Jira. Receives Ariba or middleware status callbacks and updates the linked Jira issue with status and PO number.

## Required Customer Configuration

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://openapi.ariba.com/api` | `ARIBA_API_URL` | Replace if the customer's Ariba region/gateway URL differs. |
| `/requisitioning/v2/prod/requisitions` | `ARIBA_PURCHASE_REQUEST_PATH` | Confirm the endpoint path for the customer's Ariba buying/procurement API. |
| `/suppliermanagement/v1/prod/supplierRequests` | `ARIBA_SUPPLIER_ONBOARDING_PATH` | Confirm the endpoint path for the customer's Ariba supplier lifecycle API. |
| `your-realm` | `ARIBA_REALM` | Replace with the customer's Ariba realm. |
| `PROC` | `SPACE` variables | Replace with the Jira project key. |
| `Approved` | `APPROVED_STATUS` variables | Replace with the exact Jira approval-complete status. |
| `customfield_10170` | Purchase request flows and status JQL | Jira field that stores the Ariba purchase request ID. Also replace `cf[10170]` in JQL. |
| `customfield_10171` | Supplier onboarding flow | Jira field that stores the Ariba supplier onboarding request ID. |
| `customfield_10172`, `customfield_10173` | Status callback flow | Jira fields that store Ariba status and purchase order number. |
| `customfield_10180` to `customfield_10185` | Purchase request create flow scripts | Jira fields for item description, quantity, supplier, commodity code, cost center, and need-by date. |
| `customfield_10190` to `customfield_10194` | Supplier onboarding flow scripts | Jira fields for supplier name, contact email, country, category, and tax ID. |

## Ariba Status Webhook Payload

After importing the template, copy the trigger URL from `SAP Ariba - Purchase Status to Jira` and configure Ariba webhooks or middleware to post status updates to iHub.

Recommended payload:

```json
{
  "aribaRequestId": "PR123456",
  "status": "Ordered",
  "purchaseOrderNumber": "4500001234",
  "message": "The purchase request was converted to a purchase order."
}
```

The flow also accepts variants such as `requestId`, `requisitionId`, `document.id`, `event.documentId`, `poNumber`, and `po.number`.

## Credentials

- `custom-header-sap-ariba-api`: custom-header credential for Ariba API calls. Configure `Authorization` with the bearer token and `apiKey` with the Ariba application key.

## Notes

- All flows are disabled by default.
- Ariba API paths vary by product, realm, and enabled API package. Review the endpoint path variables against the customer's Ariba API entitlement before enabling the flows.
- The create flows use Jira issue custom fields directly in scripts. Replace the sample field IDs with the fields used by the customer's JSM request types.

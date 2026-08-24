# Jira <-> Azure DevOps Sync

This template installs two iHub flows that keep Jira issues and Azure DevOps work items linked in both directions.

- `Azure Devops Outgoing Sync`: Jira -> Azure DevOps, triggered by Jira webhooks.
- `Azure Devops Incoming Sync`: Azure DevOps -> Jira, triggered by Azure DevOps Service Hooks calling an iHub custom webhook URL.

The sync stores the Azure DevOps work item ID in a Jira custom field. That field is the link key used by both flows to decide whether a Jira issue already has a matching Azure work item.

## What The Template Syncs

From Jira to Azure DevOps:

- Creates an Azure DevOps work item when a Jira issue is created and the Jira issue does not already have an Azure work item ID.
- Stores the created Azure work item ID back on the Jira issue.
- Updates the Azure work item description when the Jira issue is updated.
- Adds Jira comments to the linked Azure work item.
- Uploads Jira attachments to Azure DevOps and attaches them to the linked work item.
- Stores synced comment and attachment IDs as Jira issue properties using keys like `ihub-<external-id>` to avoid duplicate processing.

From Azure DevOps to Jira:

- Searches Jira for an issue whose Azure work item ID custom field matches the incoming Azure work item ID.
- Creates a Jira issue if Azure DevOps sends a `workitem.created` event and no matching Jira issue exists.
- Adds Azure DevOps comments to the matching Jira issue.
- Downloads Azure DevOps attachments and uploads them to the matching Jira issue.
- Stores synced attachment IDs as Jira issue properties using keys like `ihub-<external-id>`.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Azure DevOps work item ID. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template                             | Where                                      | Customer action                                                                                                                                |
| --------------------------------------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `https://dev.azure.com/rixterdev`             | Outgoing flow variable `AZURE_URL`         | Replace with the customer's Azure DevOps organization URL, for example `https://dev.azure.com/customer-org`.                                   |
| `Risk%20Management`                           | Outgoing flow variable `PROJECT_NAME`      | Replace with the target Azure DevOps project name. URL-encode spaces as `%20`.                                                                 |
| `Risk`                                        | Outgoing flow variable `WORKITEM_TYPE`     | Replace with the Azure DevOps work item type to create, such as `Task`, `Bug`, `User Story`, or a custom type. URL-encode the value if needed. |
| `AZURE`                                       | Incoming flow variable `SPACE`             | Replace with the Jira project key where Azure-created work items should create/find issues.                                                    |
| `customfield_10153`                           | Both flow JSON files                       | Replace with the customer's Jira custom field key, for example `customfield_12345`.                                                            |
| `cf[10153]`                                   | Incoming flow JQL search                   | Replace with the numeric custom field ID, for example `cf[12345]`.                                                                             |
| `Task`                                        | Incoming `Create Issue` action             | Replace with the Jira issue type to create if the target project does not use `Task`.                                                          |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. This prevents circular comment and attachment sync.                        |

## Places To Update The Jira Custom Field

Update all references to the work item ID field, not only the create action.

In `azure_devops_outgoing_sync.json`:

- `Create WorkItem in Azure` condition: checks `$.issue.fields.customfield_10153` is empty before creating a work item.
- `Comment Work Item` URL, flow variable, and condition: uses `issue.fields.customfield_10153` to comment on the linked work item.
- `Update Description on Work Item` URL, flow variable, and condition: uses `issue.fields.customfield_10153` to patch the linked work item.
- `Edit Issue on Create` body: writes the created Azure work item ID back to `customfield_10153`.

In `azure_devops_incoming_sync.json`:

- `Check if issue exists` JQL: searches `cf[10153]` for the incoming Azure work item ID.
- `Create Issue` body: writes the incoming Azure work item ID into `customfield_10153`.

## Azure DevOps Service Hook Setup

After the template is instantiated in iHub, copy the trigger URL from the `Azure Devops Incoming Sync` custom webhook trigger.

In Azure DevOps:

1. Open the Azure DevOps project.
2. Go to `Project Settings` -> `Service Hooks`.
3. Create a subscription.
4. Choose `Web Hooks`.
5. Create one subscription for `Work item created`.
6. Paste the iHub trigger URL into the webhook URL field.
7. Leave optional settings blank unless the customer wants to filter specific work item types.
8. Repeat the setup for `Work item commented on`, using the same iHub trigger URL.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. This is required because incoming Azure events create Jira comments/attachments, which would otherwise trigger the Jira -> Azure flow again and create a sync loop.

Make sure the account ID in those conditions belongs to the customer iHub/Jira integration user.

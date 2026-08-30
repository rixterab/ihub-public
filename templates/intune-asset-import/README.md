# Microsoft Intune Asset Import

Imports Microsoft Intune managed devices into Jira Assets on a daily schedule.

- `Intune Managed Device Asset Import`: creates or refreshes a Jira Assets external import source and populates a `Managed Devices` object type from Microsoft Graph.

The flow runs on `0 5 * * *` and is disabled by default. It complements `microsoft-entra-asset-import`, which covers users and groups but not endpoints.

## What The Template Does

Same seven-action shape as `microsoft-entra-asset-import`:

| Order | Action | Purpose |
| --- | --- | --- |
| 100 | Update Import Mapping | `PATCH` the import source schema and field mapping |
| 300 | Check Import Status | Read `configstatus` |
| 400 | Cancel Stuck Import | `DELETE {{cancelUrl}}` when the previous run is still `RUNNING` |
| 500 | Start Import | `POST` a new execution |
| 600 | Fetch Intune Managed Devices | `GET /deviceManagement/managedDevices`, following `@odata.nextLink` |
| 200 | Submit Data | `POST` each page to `{{submitResultsUrl}}` |
| 700 | Complete Import | `POST {"completed": true}` |

Pagination is `RESPONSE_BASED` on `@odata.nextLink`, exactly as in the Entra template, so every page is fetched and submitted.

## Imported Fields

The `Managed Devices` object type is labelled by `deviceName` and keyed by the Graph `id` (`externalIdPart`). `Operating System` and `Compliance State` are imported as referenced object types.

`id`, `deviceName`, `azureADDeviceId`, `serialNumber`, `model`, `manufacturer`, `osVersion`, `managedDeviceOwnerType`, `managementState`, `enrolledDateTime`, `lastSyncDateTime`, `userPrincipalName`, `emailAddress`, `userDisplayName`, `isEncrypted`, `jailBroken`, `totalStorageSpaceInBytes`, `freeStorageSpaceInBytes`, plus `operatingSystem` and `complianceState` as references.

The fetch sends `$top=100` and no `$select`, because `managedDevices` has historically been unreliable with `$select`. The full device object is submitted and Jira Assets picks the mapped fields; that costs bandwidth, not correctness.

Linking a device to its Entra user is possible by adding a `referenced_object` attribute against `Users` with `objectMappingIQL` on `userPrincipalName`, provided `microsoft-entra-asset-import` has already populated that object type in the same schema.

## Credentials

- `oauth2-microsoft-intune` (OAuth2, `client_credentials`): an Entra app registration with the **application** permission `DeviceManagementManagedDevices.Read.All`, admin-consented in the tenant. Replace `{tenant}` in the authorize and token URLs with the tenant ID. App-only is correct here: an unattended import should not depend on a signed-in user.
- `token-intune-asset-import-token` (Token): Jira Assets API token used for the import source calls.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `Microsoft Intune` | `assetSchema` flow variable | Replace with the target object schema name in Jira Assets. |
| `7a25e9d3-04c1-4f87-b3e6-8d95207caf12` | `importSourceId` flow variable | Replace with the customer's import source ID, or leave as-is on a fresh install to have one created. |
| `{tenant}` | `oauth2-microsoft-intune` credential URLs | Replace with the Entra tenant ID. |

## Limitations

- Managed devices only. Intune apps, configuration profiles, compliance policies and device categories are not imported.
- Devices only reachable through the Graph **beta** endpoint (some hardware detail, some Autopilot data) are out of scope; the flow uses `v1.0`.
- One-way. Nothing is written back into Intune.
- Graph throttles `managedDevices` on large tenants. The fetch has no `ratelimit` block; add one if a tenant with tens of thousands of devices starts returning `429`.

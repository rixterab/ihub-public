# Jamf Asset Import

Imports Jamf Pro computer inventory into Jira Assets on a daily schedule.

- `Jamf Computer Asset Import`: creates or refreshes a Jira Assets external import source and populates a `Computers` object type from the Jamf Pro API.

The flow runs on `0 4 * * *` and is disabled by default.

## What The Template Does

Following the same seven-action shape as `microsoft-entra-asset-import`:

| Order | Action | Purpose |
| --- | --- | --- |
| 100 | Update Import Mapping | `PATCH` the import source schema and field mapping |
| 300 | Check Import Status | Read `configstatus` |
| 400 | Cancel Stuck Import | `DELETE {{cancelUrl}}` when the previous run is still `RUNNING` |
| 500 | Start Import | `POST` a new execution, yielding `submitResultsUrl` and `cancelUrl` |
| 600 | Fetch Jamf Computers | `GET` computer inventory |
| 200 | Submit Data | `POST` the page to `{{submitResultsUrl}}` |
| 700 | Complete Import | `POST {"completed": true}` |

## Imported Fields

The `Computers` object type is labelled by `name` and keyed by Jamf's `id` (`externalIdPart`). `Department` and `Building` are imported as referenced object types, mirroring how `microsoft-entra-asset-import` handles `Office Location`.

| Attribute | Jamf locator |
| --- | --- |
| `name` (label) | `general.name` |
| `id` (external ID) | `id` |
| `udid` | `udid` |
| `serialNumber` | `hardware.serialNumber` |
| `assetTag` | `general.assetTag` |
| `model`, `modelIdentifier` | `hardware.model`, `hardware.modelIdentifier` |
| `osName`, `osVersion`, `osBuild` | `operatingSystem.*` |
| `processorType`, `totalRamMegabytes`, `macAddress` | `hardware.*` |
| `username`, `realName`, `email` | `userAndLocation.*` |
| `managed`, `supervised` | `general.remoteManagement.managed`, `general.supervised` |
| `lastContactTime`, `lastReportDate` | `general.lastContactTime`, `general.reportDate` |
| `department`, `building` | referenced objects on `userAndLocation.*` |

The fetch requests `section=GENERAL&section=HARDWARE&section=OPERATING_SYSTEM&section=USER_AND_LOCATION`. Adding an attribute from another section means adding its `section` to the URL as well, otherwise the locator resolves to nothing.

## Credentials

- `oauth2-jamf-pro` (OAuth2, `client_credentials`): calls the Jamf Pro API. In Jamf go to `Settings` -> `API roles and clients`, create an **API role** with the `Read Computers` privilege, then an **API client** granted that role. Use its client ID and client secret. Replace `{instance}` in the credential's `accessTokenUrl` with the customer's Jamf Cloud instance name.
- `token-jamf-asset-import-token` (Token): Jira Assets API token used for the import source calls.

Jamf's older `/api/v1/auth/token` basic-auth flow also works but issues 30-minute tokens tied to a named user; the API client is the right mechanism for an unattended import.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://your-instance.jamfcloud.com` | `JAMF_URL` flow variable | Replace with the Jamf Pro base URL, no trailing slash. |
| `Jamf` | `assetSchema` flow variable | Replace with the target object schema name in Jira Assets. |
| `0f3c81a4-7d26-4b95-9e10-52c8a7b3f640` | `importSourceId` flow variable | Replace with the customer's import source ID, or leave as-is on a fresh install to have one created. |
| `{instance}` | `oauth2-jamf-pro` credential token URL | Replace with the Jamf Cloud instance name. |
| `jamf.svg` | Template folder | Placeholder mark, not the Jamf logo. Replace with the official asset before publishing. |

## Limitations

- **No pagination.** The fetch requests a single page of `page-size=1000` sorted by ID. Instances with more than 1000 computers silently import only the first page. Adding pagination means a `RESPONSE_BASED` pagination variable on the fetch action; `microsoft-entra-asset-import` is the working reference, but Jamf pages by `page` number rather than returning a next-page URL, so the variable has to construct the URL itself.
- **Computers only.** Mobile devices (`/api/v2/mobile-devices`) and users are not imported. A second flow following the same shape would cover them.
- One-way. Nothing is written back into Jamf.
- The import replaces the object set each run; objects deleted in Jamf are handled by the import source's own deletion policy, configured in Jira Assets rather than here.

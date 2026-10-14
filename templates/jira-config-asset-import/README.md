# Jira Config Asset Import

This template installs one scheduled iHub flow, `Import Jira Configuration into Assets`, that reads the configuration of the Jira Cloud site iHub is installed on and imports it into Jira Assets as an external import. Projects, work item types, workflows, schemes, screens, fields, filters, users and groups become Assets objects, linked to each other where the Jira API returns an ID to link on.

It is **one-way**: Jira configuration -> Jira Assets. Nothing in Assets is written back to Jira, and editing an imported object does not change Jira.

The object types follow the Jira Data Center "Jira Config" schema (the one exported to `insight_schemas_*.json`) wherever Jira Cloud has an equivalent. [What Is Not Imported](#what-is-not-imported) lists every Data Center type that was left out and why.

Use it to answer configuration-governance questions with AQL instead of clicking through Jira admin: which custom fields have not been used for a year, which projects have no category or a deactivated lead, which statuses exist but are not on any workflow, who owns the filters that everyone has favourited.

## What The Template Does

The flow runs daily at `0 3 * * *` and is disabled by default. It follows the standard Assets external-import sequence (update the mapping, clear any stuck run, start an execution, submit data, complete), with one fetch-and-submit pair per Jira object type:

| Order | Action | Purpose |
| --- | --- | --- |
| 100 | Update Import Mapping | `PATCH` the import source with the schema and field mapping for all 20 object types |
| 200 | Check Import Status | Read `configstatus` |
| 300 | Cancel Stuck Import | `DELETE {{cancelUrl}}` when the previous run is still `RUNNING` |
| 400 | Start Import | `POST` a new execution, yielding `submitResultsUrl` and `cancelUrl` |
| 500-4400 | Fetch Jira `<type>` / Submit Jira `<type>` | 20 pairs. Each fetch reads one Jira REST endpoint, paging by `startAt` where the endpoint is paginated. Its child submits each page to `{{submitResultsUrl}}` under its own selector |
| 4500 | Complete Import | `POST {"completed": true}` so Assets processes everything submitted |

All fetches are children of `Start Import` and run in `order`, before `Complete Import`. All 20 object types go through one import source and one execution, so the customer configures one import source and one token, not twenty.

## Imported Object Types

Every object type is labelled by `Name`. The Jira ID (or account ID, group ID or application key) is the `externalIdPart`, so renaming something in Jira updates the existing Assets object instead of creating a new one.

Assets reserves `Key`, `Name`, `Created` and `Updated` as system attributes, so Jira's own values use `Project Key`, `Created Date` and `Updated Date`.

| Object type | Jira endpoint | Attributes | References |
| --- | --- | --- | --- |
| Project Category | `GET /rest/api/3/projectCategory` | Jira ID, Description | |
| User | `GET /rest/api/3/users/search` | Account ID, Account Type, Active, Email Address, Time Zone, Locale | |
| Group | `GET /rest/api/3/group/bulk` | Group ID | |
| Application | `GET /rest/api/3/applicationrole` | Application Key, User Count, Number Of Seats, Remaining Seats, Unlimited Seats, Platform | |
| Project | `GET /rest/api/3/project/search?expand=description,lead,url,insight` | Jira ID, Project Key, Project Type, Team Managed, Private, Description, URL, Issue Count, Last Issue Update | Lead -> User, Category -> Project Category |
| Issue Type | `GET /rest/api/3/issuetype` | Jira ID, Description, Hierarchy Level, Subtask, Scope, Icon URL | Project -> Project (team-managed types only) |
| Issue Type Scheme | `GET /rest/api/3/issuetypescheme` | Jira ID, Description, Default | Default Issue Type -> Issue Type |
| Status | `GET /rest/api/3/statuses/search` | Jira ID, Description, Status Category, Scope | Project -> Project (team-managed statuses only) |
| Priority | `GET /rest/api/3/priority` | Jira ID, Description, Default, Color, Icon URL | |
| Resolution | `GET /rest/api/3/resolution/search` | Jira ID, Description, Default | |
| Field | `GET /rest/api/3/field/search?expand=lastUsed,screensCount,contextsCount,isLocked,searcherKey` | Jira ID, Description, Field Type, Custom Field Type, Type Name, Locked, Screens Count, Contexts Count, Last Used, Last Used Tracking | |
| Workflow | `GET /rest/api/3/workflows/search` | Jira ID, Description, Scope, Editable, Version, Created Date, Updated Date | Project -> Project (team-managed workflows only) |
| Workflow Scheme | `GET /rest/api/3/workflowscheme` | Jira ID, Description, Default Workflow, Draft, Last Modified | |
| Screen | `GET /rest/api/3/screens` | Jira ID, Description, Scope | Project -> Project (team-managed screens only) |
| Screen Scheme | `GET /rest/api/3/screenscheme` | Jira ID, Description | Default / Create / Edit / View Screen -> Screen |
| Issue Type Screen Scheme | `GET /rest/api/3/issuetypescreenscheme` | Jira ID, Description | |
| Notification Scheme | `GET /rest/api/3/notificationscheme` | Jira ID, Description | |
| Permission Scheme | `GET /rest/api/3/permissionscheme` | Jira ID, Description | |
| Project Role | `GET /rest/api/3/role` | Jira ID, Description, Admin, Default, Scope | |
| JQL Filter | `GET /rest/api/3/filter/search?overrideSharePermissions=true&expand=…` | Jira ID, Description, JQL, Favourite Count, Last Used, URL | Owner -> User |

References are resolved with `objectMappingIQL` against the referenced type's ID attribute, for example `"Jira ID" = ${projectCategory.id}` and `"Account ID" = ${lead.accountId}`, not by name. Jira allows duplicate names for team-managed statuses, work item types and workflows, so a name match would link to the wrong object. Object types are declared in dependency order (Project Category and User before Project, Project before Issue Type, Screen before Screen Scheme).

`Workflow Scheme.Default Workflow` is plain text, not a reference. Jira returns the workflow's name there, not its ID, and workflow names are not unique across team-managed projects.

## API Calls

`Jira` is iHub's own Jira connection (`ATLASSIAN_TOKEN`). `Admin` is the `basic-auth-jira-admin` credential, a Jira administrator's email and API token (see [Credential](#credential)). `Import` is the `token-jira-config-asset-import-token` credential.

| Action | Method | Endpoint | Credential | Paginated |
| --- | --- | --- | --- | --- |
| Update Import Mapping | `PATCH` | `https://api.atlassian.com/jsm/assets/workspace/{{context.workspaceId}}/v1/importsource/{{_flow.importSourceId}}/mapping` | Import | |
| Check Import Status | `GET` | `…/importsource/{{_flow.importSourceId}}/configstatus` | Import | |
| Cancel Stuck Import | `DELETE` | `{{cancelUrl}}` | Import | |
| Start Import | `POST` | `…/importsource/{{_flow.importSourceId}}/executions` | Import | |
| Fetch Jira Project Categories | `GET` | `{{baseUrl}}/rest/api/3/projectCategory` | Jira | No, plain array |
| Fetch Jira Users | `GET` | `{{baseUrl}}/rest/api/3/users/search?maxResults=1000&startAt={{page}}` | Jira | `startAt` += 1000, plain array |
| Fetch Jira Groups | `GET` | `{{baseUrl}}/rest/api/3/group/bulk?maxResults=250&startAt={{page}}` | Jira | `startAt` += 250 |
| Fetch Jira Applications | `GET` | `{{baseUrl}}/rest/api/3/applicationrole` | Admin | No, plain array |
| Fetch Jira Projects | `GET` | `{{baseUrl}}/rest/api/3/project/search?maxResults=250&expand=…&startAt={{page}}` | Jira | `startAt` += 250 |
| Fetch Jira Issue Types | `GET` | `{{baseUrl}}/rest/api/3/issuetype` | Jira | No, plain array |
| Fetch Jira Issue Type Schemes | `GET` | `{{baseUrl}}/rest/api/3/issuetypescheme?maxResults=50&startAt={{page}}` | Admin | `startAt` += 250, see [Limitations](#limitations) |
| Fetch Jira Statuses | `GET` | `{{baseUrl}}/rest/api/3/statuses/search?maxResults=200&startAt={{page}}` | Admin | `startAt` += 200 |
| Fetch Jira Priorities | `GET` | `{{baseUrl}}/rest/api/3/priority?startAt={{page}}&maxResults=200` | Jira | Plain array, see [Limitations](#limitations) |
| Fetch Jira Resolutions | `GET` | `{{baseUrl}}/rest/api/3/resolution/search?maxResults=50&startAt={{page}}` | Jira | `startAt` += 50 |
| Fetch Jira Fields | `GET` | `{{baseUrl}}/rest/api/3/field/search?expand=…&maxResults=200&startAt={{page}}` | Admin | `startAt` += 200 |
| Fetch Jira Workflows | `GET` | `{{baseUrl}}/rest/api/3/workflows/search?maxResults=50&startAt={{page}}` | Admin | `startAt` += 50 |
| Fetch Jira Workflow Schemes | `GET` | `{{baseUrl}}/rest/api/3/workflowscheme?maxResults=50&startAt={{page}}` | Admin | `startAt` += 50 |
| Fetch Jira Screens | `GET` | `{{baseUrl}}/rest/api/3/screens?maxResults=100&startAt={{page}}` | Admin | `startAt` += 100 |
| Fetch Jira Screen Schemes | `GET` | `{{baseUrl}}/rest/api/3/screenscheme?maxResults=100&startAt={{page}}` | Admin | `startAt` += 100 |
| Fetch Jira Issue Type Screen Schemes | `GET` | `{{baseUrl}}/rest/api/3/issuetypescreenscheme?maxResults=100&startAt={{page}}` | Admin | `startAt` += 100 |
| Fetch Jira Notification Schemes | `GET` | `{{baseUrl}}/rest/api/3/notificationscheme?maxResults=50&startAt={{page}}` | Admin | `startAt` += 50 |
| Fetch Jira Permission Schemes | `GET` | `{{baseUrl}}/rest/api/3/permissionscheme` | Jira | No, `{"permissionSchemes": [...]}` |
| Fetch Jira Project Roles | `GET` | `{{baseUrl}}/rest/api/3/role` | Admin | No, plain array |
| Fetch Jira JQL Filters | `GET` | `{{baseUrl}}/rest/api/3/filter/search?maxResults=50&startAt={{page}}&overrideSharePermissions=true&expand=…` | Admin | `startAt` += 50 |
| Submit Jira `<type>` (×20) | `POST` | `{{submitResultsUrl}}` | Import | |
| Complete Import | `POST` | `{{submitResultsUrl}}` | Import | |

Things worth knowing about the Jira side:

- **The administrator-only endpoints run as `Admin`.** Statuses, workflows, workflow schemes, screens, screen schemes, work item type schemes, work item type screen schemes, notification schemes, project roles, application roles, fields and filters with `overrideSharePermissions` need *Administer Jira*. Those twelve fetches use the `basic-auth-jira-admin` credential, so they see the site the way that administrator does. The other eight use iHub's own connection: users and groups need *Browse users and groups*, and projects, work item types, priorities, resolutions and permission schemes return whatever that connection can see.
- **`/rest/api/3/status` is deliberately not used.** It returns only statuses that are on an active workflow. `/rest/api/3/statuses/search` returns every status, including unused ones, which are exactly what a cleanup review is looking for. Its `statusCategory` is a string (`TODO`, `IN_PROGRESS`, `DONE`), so Status Category is a text attribute here, not a separate object type as in Data Center.
- **`/rest/api/3/workflow/search` is not used either.** Atlassian deprecated it and scheduled its removal for 1 June 2026. The flow calls its replacement, `/rest/api/3/workflows/search`, which covers both company-managed (`GLOBAL`) and team-managed (`PROJECT`) workflows.
- **Some parameters are marked experimental by Atlassian:** `expand=insight` on project search (Issue Count, Last Issue Update), `approximateLastUsed` and `overrideSharePermissions` on filter search. If Atlassian removes them, those attributes come back empty. The rest of the import is unaffected.
- **Pagination is `AUTO_INCREMENT`.** Each paginated fetch has a `page` variable that starts at `0` and increases by a fixed step on every request, and the URL passes it as `startAt={{page}}`. Paging stops when `breakJsonPath` comes back empty: `values` for the paged endpoints, the bare array for users, and `nextPage` for work item type schemes. The step must equal the number of items Jira actually returns per page. If Jira returns fewer than the step, the items in between are skipped without any error (see [Limitations](#limitations)).
- Each Jira fetch has a `ratelimit` block with `maxRetries: 3` on `408/429/5xx`. These are idempotent `GET`s, so retrying is safe, and a type that fails to fetch has worse consequences here than a slow run (see [Limitations](#limitations)).

## Body Format And Fidelity

- **Each Submit action sends the fetched page unchanged**, as `{{{toJSON payload}}}` for plain-array responses or `{{{toJSON payload.values}}}` for paged ones, under that type's selector. IDs go to Assets as Jira returns them: numeric for workflow schemes, screens, screen schemes, notification schemes, permission schemes and project roles, and strings for the rest.
- **Every account type is imported as a User.** People (`atlassian`), app and bot accounts (`app`) and JSM portal customers (`customer`) all become objects, and on a service-desk site customers can outnumber staff many times over. The `USER_ACCOUNT_TYPES` flow variable is meant to narrow this, but no action reads it at the moment (see [Limitations](#limitations)).
- **Email Address is often empty.** Jira applies each user's profile visibility settings, so most addresses are hidden from the API unless the user or the organisation has made them visible.
- **Timestamps are text attributes**, imported as Jira returns them (ISO 8601): Last Issue Update, Last Used, Created Date, Updated Date, Last Modified. This avoids the import failing on a date format that does not parse. Text ISO timestamps still sort correctly. To filter on them as dates in AQL, change the attribute `type` to `date` in the mapping body and test against the site.
- **Descriptions and JQL are `text` attributes**, which Assets limits to 255 characters. A longer project description or JQL query may be cut off or rejected for that object, depending on how Assets handles it. No template in this repository uses `textarea`, so this one does not either.
- URLs (`URL`, `Icon URL`) are `text`, not `url`, for the same reason: `url` is not used anywhere else in this repository.

## Credential

The flow uses three credentials. The [API Calls](#api-calls) table shows which action uses which.

**iHub's Jira connection (`ATLASSIAN_TOKEN`)** handles project categories, users, groups, projects, work item types, priorities, resolutions and permission schemes. There is nothing to paste.

**`basic-auth-jira-admin` (`BASIC_AUTH`)** handles the twelve fetches that need *Administer Jira*. It is the email address and API token of a Jira administrator:

| Field | Value to paste |
| --- | --- |
| Username | The administrator's Atlassian account email, for example `jira-admin@example.com` |
| Password | An API token created at <https://id.atlassian.com/manage-profile/security/api-tokens>, pasted as-is |

Create an **API token without scopes** (the classic kind). The fetches call the site URL (`{{baseUrl}}`), and scoped API tokens only work through the `api.atlassian.com/ex/jira/<cloudId>` gateway, so a scoped token is rejected with `401`. Use a dedicated administrator or service account rather than a person's own account. The import runs as that account, and it fails when the person leaves or the token expires (see [Limitations](#limitations)).

**`token-jira-config-asset-import-token` (`TOKEN`)** is the Jira Assets external import token. iHub sends it as `Authorization: Bearer <token>`, so paste the **raw token, without a `Bearer ` prefix**.

To create it: in Jira Assets, open the target object schema, go to **Schema configuration -> Import -> Create import -> External import**, name it (for example `Jira Config`), and generate a token.

## Jira Assets Setup

1. Create an empty object schema in Jira Assets, for example `Jira Config`. Its name goes in `assetSchema`.
2. In that schema, create an **External import** and generate its token (see [Credential](#credential)).
3. Find the import source ID: call `GET https://api.atlassian.com/jsm/assets/v1/imports/info` with `Authorization: Bearer <token>`. The `links` in the response contain `…/workspace/<workspaceId>/v1/importsource/<importSourceId>/…`. Put `<importSourceId>` in the `importSourceId` flow variable. iHub supplies the workspace ID itself as `{{context.workspaceId}}`.
4. In the import's settings, decide what happens to objects that are missing from a run (the import source's missing-objects policy). A disabled or deleted Jira item then disappears from Assets on the next run, or is kept and marked, depending on that choice. Read the first item under [Limitations](#limitations) before choosing to delete.
5. Make sure the `basic-auth-jira-admin` account has *Administer Jira*, and that iHub's Jira connection has *Browse users and groups* (see [API Calls](#api-calls)).
6. Install the template, set the flow variables and both credentials, then run the flow once manually. `Update Import Mapping` creates the 20 object types and their attributes in the schema on the first run.
7. Enable the flow.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `Jira Config` | `assetSchema` flow variable | Name of the target object schema in Jira Assets. |
| `00000000-0000-0000-0000-000000000000` | `importSourceId` flow variable | Replace with the import source ID from step 3 of [Jira Assets Setup](#jira-assets-setup). |
| `atlassian` | `USER_ACCOUNT_TYPES` flow variable | Intended as a comma-separated list of account types to import as User objects (`atlassian`, `app`, `customer`). No action reads it yet, so changing it has no effect. |
| `jira-admin@example.com` | `basic-auth-jira-admin` credential, username | Email address of the Jira administrator account the admin-only fetches run as. |
| API token | `basic-auth-jira-admin` credential, password | That account's unscoped Atlassian API token, pasted as-is. |
| Token value | `token-jira-config-asset-import-token` credential | The raw Jira Assets external import token, no `Bearer ` prefix. |
| `0 3 * * *` | Flow trigger | Change the schedule if daily at 03:00 does not suit. Jira configuration changes slowly; more often than daily is rarely useful. |

There are no Jira custom fields to configure. The flow reads configuration and does not write to any work item.

## Security

This flow has no webhook trigger. There is no inbound URL to protect.

`basic-auth-jira-admin` holds a Jira administrator's API token. It can do anything that administrator can do, through any Jira API, not just read the endpoints this flow calls. Treat it as an admin password: use a dedicated account, keep it out of tickets and chat, and revoke the token at id.atlassian.com if it leaks.

The data it imports is sensitive in a different way. The imported schema shows the whole site's configuration, every user's account ID and any visible email address, and, because of `overrideSharePermissions=true`, the name, owner and **JQL of private filters** that their owners never shared. Anyone with access to the object schema can read all of it. Restrict the schema's roles in Assets to Jira administrators, or remove `overrideSharePermissions=true` from the filter fetch URL so that only filters shared with the `basic-auth-jira-admin` account are imported.

## What Is Not Imported

These Data Center "Jira Config" object types have no Cloud counterpart in this template:

| Data Center type | Why not |
| --- | --- |
| Component, Version | Cloud only lists them per project (`/project/{key}/version`, and `/rest/api/3/component` does not say which project a component belongs to). Importing them needs an iterator over projects. |
| Role (actors per project) | `GET /rest/api/3/project/{key}/role/{id}`: one call per project per role. |
| Screen Tab, Screen Tab Field | One call per screen, then one per tab. |
| Screen - Issue Operation | Partly covered: Screen Scheme carries Default / Create / Edit / View Screen references. |
| Screen Scheme - Issue Type, Workflow - Issue Type | Bulk mapping endpoints exist (`/rest/api/3/issuetypescreenscheme/mapping`, `/rest/api/3/issuetypescheme/mapping`), and they are the next thing to add. Workflow scheme `issueTypeMappings` is a JSON map, not an array, and needs a scripted variable to flatten. |
| Field Configuration, Field Configuration Scheme, Field Configuration Field, Field Configuration - Issue Type | Deprecated in Cloud in favour of field schemes (`/rest/api/3/config/fieldschemes`), and that API is still marked experimental. Neither is imported until it settles. |
| Notification Event, Permission Access, Project Permissions | Nested inside schemes (`expand=all` / `expand=permissions`), or returned as a JSON map (`/rest/api/3/permissions`) rather than an array, so each needs a flattening script. |
| Event Type | Available at `/rest/api/3/events`. Left out because the notification schemes are the useful view of it. |
| License | Cloud's `/rest/api/3/instance/license` returns only a plan per product. Application covers seats and user counts. |
| Plugin | Jira Cloud has no supported public REST API listing installed apps. |
| User Directory, Application Link | Data Center only; managed at the Atlassian organisation level in Cloud. |
| Status Category | Imported as a text attribute on Status (see [API Calls](#api-calls)). |
| Applications, Projects, Issues, Workflows, Screens, Fields, Notifications, Permissions, Users, System | Empty parent types that Data Center used to group the schema into a tree. External imports here create a flat list of object types. No template in this repository uses nested object types in an import mapping, so this one does not either. |

## Limitations

- **A fetch that fails can remove objects.** If one fetch returns `401` or `403`, or keeps failing after retries, its submit never runs, but `Complete Import` still does. Assets then treats every object of that type as missing from this run and applies the import source's missing-objects policy. With a delete policy, a permission problem deletes that object type from Assets on the next run. The most likely causes are an expired or revoked `basic-auth-jira-admin` token, or that account losing *Administer Jira*, which takes out twelve object types at once. Atlassian API tokens expire (one year at most), so put the expiry date in a calendar. Run the flow once manually and check every fetch in the flow log before enabling a delete policy.
- **Paging assumes Jira returns a full page.** `AUTO_INCREMENT` adds a fixed step to `startAt` and cannot read how many items the previous page really held. If Jira returns fewer items than the step, because it caps `maxResults` below the requested value, the items in between are skipped without any error. Two cases need checking:
  - `Fetch Jira Issue Type Schemes` asks for `maxResults=50` but steps by 250. It also stops on `nextPage`, so a site with more than 50 work item type schemes imports only the first 50. Set the step to 50 to match.
  - `Fetch Jira Projects` and `Fetch Jira Groups` ask for 250 per page, and `Fetch Jira Fields` for 200. If Jira caps any of these lower, the step must come down to the cap. Compare the `total` in the first response with the number of objects imported.
- **Priorities use the deprecated `/rest/api/3/priority`.** It returns a plain array and ignores `startAt`, while the pagination block stops on `values`, which that response does not have. Whether iHub treats a missing break path as "stop" or keeps requesting the same array has not been checked. `/rest/api/3/priority/search` returns `values` and `isLast`, and it is the endpoint Atlassian supports.
- **Group membership is not imported.** Group has name and ID only. Membership is `GET /rest/api/3/group/member` per group, which needs an iterator over groups.
- **References use unquoted IDs.** `${…}` in `objectMappingIQL` is unquoted, following the repository's precedent (`Name Like ${officeLocation}`), but Atlassian account IDs can contain `:` (`712020:…`). If Lead or Owner references come back empty, try `"Account ID" = "${lead.accountId}"`.
- **Read-only snapshot.** No change history is kept beyond what Assets records on each object. Changing an imported object in Assets does not touch Jira.
- **Submit volume grows with the site.** Each run makes 20 submits plus one per extra page. A site with thousands of fields or filters makes proportionally more. The Assets actions carry no `ratelimit` block, matching the other asset-import templates.
- **Freshness is the schedule.** The flow reflects Jira as of its last run. It does not react to configuration changes as they happen.

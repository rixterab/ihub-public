# Jira Service Management - Create Employee in Personio

This template installs one iHub flow that creates a new employee in Personio when a Jira Service Management onboarding request is approved.

- `Jira Service Management - Create Employee in Personio`: Jira -> Personio, triggered by Jira webhooks.

The integration is **one-way**. Jira is where the request is raised and approved; once it is approved, Personio becomes the system of record and nothing flows back from Personio except the new employee ID and a result comment on the request. See [Limitations](#limitations).

Two related templates cover the rest of the employee lifecycle in the same direction: `personio-employee-update-from-jira` (changes to an existing employee) and `personio-absence-from-jira` (approved leave). The `personio` template covers the opposite direction (Personio events -> JSM onboarding, offboarding and mover requests); see [Loops Across Templates](#loops-across-templates) before installing both.

## What The Template Does

In execution order:

1. Listens for Jira `avi:jira:updated:issue` events.
2. Ignores everything except the update that **moves** a request of the configured project, work item type and (optionally) JSM request type **into** the approved status. Later edits of an already approved request, and iHub's own write-back, never qualify. See [How Duplicate Processing Is Prevented](#how-duplicate-processing-is-prevented).
3. Ignores requests that already hold a Personio employee ID.
4. Re-reads the request from Jira, so a late redelivery of an event for a request that has since moved on is ignored.
5. Validates the mapped Jira fields. Missing first name, last name, email or start date, an invalid email, a start date that is not a date, non-numeric weekly hours, or a department/office/legal entity that has no entry in its mapping each stop the flow with an internal comment listing every problem.
6. Searches Personio for an employee with the same email. If one exists, the request is linked to that employee (its ID is written to the Jira field) and a comment says no second employee was created.
7. Resolves the supervisor's email to a Personio person ID. If a supervisor was requested but is not in Personio, the flow stops with a comment.
8. Creates the person and their first employment in Personio. Optional values (position, department, office, legal entity, weekly hours, supervisor) are only sent when the request supplies them.
9. On success, writes the Personio employee ID into the configured Jira field and comments with the result. On failure, comments with Personio's error title and detail.

All comments are JSM internal notes by default (`JIRA_COMMENTS_INTERNAL`).

## Example Jira Workflow

```
Open -> Waiting for approval -> Approved -> In progress -> Done
                 |
                 +-> Declined
```

Configure a JSM approval step on `Waiting for approval` that transitions to `Approved` when approval completes. The flow runs on that transition. Because it acts on the transition and not on the request being created, nothing happens in Personio until an approver has said yes.

To retry a request after correcting it, move it out of `Approved` (for example back to `Waiting for approval`) and into `Approved` again.

## Required Jira Fields

| Data | Flow variable | Required | Jira field type | Personio field |
| --- | --- | --- | --- | --- |
| First name | `JIRA_FIRST_NAME_FIELD` | Yes | Short text | `first_name` |
| Last name | `JIRA_LAST_NAME_FIELD` | Yes | Short text | `last_name` |
| Work email | `JIRA_EMAIL_FIELD` | Yes | Short text | `email` (unique in Personio) |
| Start / hire date | `JIRA_START_DATE_FIELD` | Yes | Date picker | `employments[0].employment_start_date` |
| Position | `JIRA_POSITION_FIELD` | No | Short text or select | `employments[0].position.title` |
| Department | `JIRA_DEPARTMENT_FIELD` | No | Select or short text | `employments[0].org_units[]` with `type: department`, via `DEPARTMENT_ID_MAP` |
| Office | `JIRA_OFFICE_FIELD` | No | Select or short text | `employments[0].office.id`, via `OFFICE_ID_MAP` |
| Weekly working hours | `JIRA_WEEKLY_HOURS_FIELD` | No | Number | `employments[0].weekly_working_hours` |
| Supervisor | `JIRA_SUPERVISOR_FIELD` | No | User picker, or short text holding an email | `employments[0].supervisor.id`, resolved by email |
| Legal entity (subcompany) | `JIRA_LEGAL_ENTITY_FIELD` | No | Select or short text | `employments[0].legal_entity.id`, via `LEGAL_ENTITY_ID_MAP` |
| Personio employee ID | `JIRA_PERSONIO_PERSON_ID_FIELD` | Yes | Short text | Written by the flow |

Every field is referenced by its **field ID** in a flow variable, for example `customfield_10231`. Nothing in the flow JSON names a custom field, so the template imports into any site unchanged. Find a field's ID under `Settings` -> `Work items` -> `Custom fields` -> the field's menu -> the ID in the URL, or with `GET /rest/api/3/field`. Built-in fields work too, by their system ID: `summary`, `reporter`, `duedate`.

Values are read with a small normaliser: a select option becomes its value, a user picker becomes the user's email address, a multi-value field becomes its first entry, and a date-time becomes its date. This is how one flow variable can point at a text field on one site and a select list on another.

Subcompany: Personio's v2 API deprecates `subcompany` in favour of `legal_entity`, so the template only sends `legal_entity`.

## Field Mappings

Personio v2 references departments, offices, legal entities and supervisors **by ID, not by name**. Jira users pick names. Three flow variables translate one into the other:

```
DEPARTMENT_ID_MAP   = Engineering=943;Sales=944;People & Culture=945
OFFICE_ID_MAP       = Stockholm=231;Berlin=232
LEGAL_ENTITY_ID_MAP = Rixter AB=465;Rixter GmbH=466
```

- Pairs are `Jira value=Personio ID`, separated by `;`. Matching is case-insensitive and ignores surrounding spaces.
- A Jira value with no entry stops the flow with a comment naming the value and the map. This is deliberate: sending an unmapped department would either be rejected by Personio or, worse, put the employee somewhere wrong.
- An **empty** map sends the Jira value unchanged as the ID. Use this when the Jira field already holds Personio IDs, for example an Assets object attribute or a select whose option values are the IDs.
- Labels cannot contain `;` or `=`.

Where to read the IDs, with the same Personio credential:

| Map | Endpoint |
| --- | --- |
| `DEPARTMENT_ID_MAP` | `GET https://api.personio.de/v2/org-units?type=department` |
| `OFFICE_ID_MAP` | `GET https://api.personio.de/v2/workplaces` (see [Limitations](#limitations)) |
| `LEGAL_ENTITY_ID_MAP` | `GET https://api.personio.de/v2/legal-entities` |

The supervisor needs no map: the flow looks the supervisor's email up in Personio with `GET /v2/persons?email=…`.

## Personio API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Confirm Onboarding Request Is Approved | `GET` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}?fields=status` (Jira) |
| Comment Invalid Onboarding Request on Jira | `POST` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/comment` (Jira) |
| Find Personio Employee by Email | `GET` | `{{_flow.PERSONIO_API_URL}}/persons?email=…&limit=1` |
| Comment Personio Lookup Failure on Jira | `POST` | Jira comment |
| Store Existing Personio Employee ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` (Jira) |
| Comment Existing Personio Employee on Jira | `POST` | Jira comment |
| Find Personio Supervisor | `GET` | `{{_flow.PERSONIO_API_URL}}/persons?email=…&limit=1` |
| Comment Personio Supervisor Not Found on Jira | `POST` | Jira comment |
| Create Personio Employee | `POST` | `{{_flow.PERSONIO_API_URL}}/persons` |
| Store Personio Employee ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` (Jira) |
| Comment Personio Employee Created on Jira | `POST` | Jira comment |
| Comment Personio Create Failure on Jira | `POST` | Jira comment |

`PERSONIO_API_URL` defaults to `https://api.personio.de/v2`, Personio's only documented host.

Things about the Personio API that shape the flow:

- **This uses Personio API v2.** `POST /v2/persons` creates the person and their first employment in one call and answers `201` with `{"id": "3003", "employments": [{"id": "c85d…"}]}`. The flow stores `$.response.data.id` in `personioPersonId` and `$.response.data.employments[0].id` in `personioEmploymentId`. The v1 `POST /v1/company/employees` endpoint still works and is not marked deprecated, but v1 authentication is being tightened (query-string credentials on `POST /v1/auth` are removed on 2026-12-04) and v2 is where Personio is adding endpoints.
- **Errors are `application/problem+json`**: `{"personio_trace_id": "…", "errors": [{"title": "Bad request", "detail": "…"}]}`. The flow captures `errors[0].title` and `errors[0].detail` and puts both in the failure comment. Only the first error is reported; the full list is in the flow log.
- **Email is unique in Personio**, but Personio documents no specific error for a duplicate. That is why the flow looks the email up first instead of relying on the create call to fail.
- **Two results are never needed here**, so the email lookup asks for `limit=1`.
- **The supervisor lookup always runs.** iHub skips an action's children when its condition is not met, so an optional lookup cannot sit in the middle of a chain. Without a supervisor the query is unfiltered (`GET /v2/persons?limit=1` returns any one person), the next step always has a result to continue from, and the create body leaves the supervisor out. That costs one read call per request without a supervisor.
- **Every Personio call sends `X-Personio-App-ID: IHUB_JIRA_INTEGRATION`.** Personio asks integrations to identify themselves with this header and says it may not be able to troubleshoot requests without it.

## Body Format And Fidelity

Every Jira value goes through a scripted variable that reads the configured field and JSON-escapes it. Values are therefore interpolated with triple braces and survive quotes, ampersands, apostrophes and backslashes: `Ava "AJ" O'Neil & Sons` arrives in Personio as typed. This is stronger than the usual repository trade-off, where a double quote in a summary can break the body.

Values that come back **from Personio** (error titles and details) are not escaped by the flow. The comment actions emit each one as its own ADF text node through `{{{toJSON …}}}`, and leave it out when it is empty, so a quote in a Personio error message cannot break the comment.

No description or rich text is copied. Personio person fields are plain strings and the request description is not mapped.

## Credential

The manifest defines one credential: `oauth2-personio`, of type `OAUTH2` with the `client_credentials` grant against `https://api.personio.de/v2/auth/token`, sending parameters as `x-www-form-urlencoded`. This is the shape `jamf-asset-import` uses for its API client, and it is what Personio requires: the token endpoint rejects JSON bodies.

1. In Personio, create API credentials. Personio's own docs name two paths depending on the UI version: `Settings` -> `Integrations` -> `API credentials`, or `Marketplace` -> `Connected integrations` -> `Create custom integration`.
2. Grant the credential these permissions:

   | Permission | Scope string | Used by |
   | --- | --- | --- |
   | Read persons | `personio:persons:read` | `Find Personio Employee by Email`, `Find Personio Supervisor` |
   | Write persons | `personio:persons:write` | `Create Personio Employee` |

   Grant only these. Personio assigns every scope of the credential when the token request names none, which is what iHub sends, so the credential's permissions are the effective limit.
3. In iHub, open the `oauth2-personio` credential and paste the **client ID** and **client secret** exactly as Personio shows them, with no prefix. iHub fetches the bearer token (prefixed `papi-`, valid 24 hours) and sends it as `Authorization: Bearer papi-…`; you never paste a token.

Personio says that adding or removing scopes on a credential invalidates its existing tokens. If you change permissions, re-authorise the iHub credential.

The same Personio credential can be reused by `personio-employee-update-from-jira` and `personio-absence-from-jira`, which add their own scopes.

## Jira-side Setup

1. Create a single-line text custom field, for example `Personio Employee ID`, and add it to the onboarding request's work item screen (it does not need to be on the portal form). Put its ID in `JIRA_PERSONIO_PERSON_ID_FIELD`.
2. Make sure the onboarding request type collects first name, last name, work email and start date, plus any optional field you want sent. Put each field's ID in the matching flow variable.
3. Add an approval step to the request type's workflow that transitions to the status in `APPROVED_STATUS_NAME`.
4. Fill in the mapping variables for department, office and legal entity, or leave them empty if the Jira fields already hold Personio IDs.
5. Put the JSM request type name in `REQUEST_TYPE_NAME` and the ID of the `Customer Request Type` field in `JIRA_REQUEST_TYPE_FIELD`. Leave `REQUEST_TYPE_NAME` empty only if no other request type in the project shares the work item type and approved status; otherwise, for example, an approved leave request would be validated as an onboarding request and get an "is empty" comment.
6. Enable the flow and approve a test request. Turn `logLevel` up while testing and back to `OFF` afterwards; the flow log then holds employee personal data.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `HR` | `SPACE` flow variable | Replace with the JSM project key. |
| `Service Request` | `REQUEST_ISSUE_TYPE` flow variable | Replace with the work item type of the onboarding request. |
| `Onboard new employee` | `REQUEST_TYPE_NAME` flow variable | Replace with the JSM request type name, or empty it (see setup step 5). |
| `replace-with-request-type-field-id` | `JIRA_REQUEST_TYPE_FIELD` flow variable | Replace with the `Customer Request Type` field ID, often `customfield_10010`. |
| `Approved` | `APPROVED_STATUS_NAME` flow variable | Replace if your approved status has a different name. |
| `replace-with-jira-field-id` | `JIRA_FIRST_NAME_FIELD`, `JIRA_LAST_NAME_FIELD`, `JIRA_EMAIL_FIELD`, `JIRA_START_DATE_FIELD` flow variables | Required. Replace with the field IDs. Until you do, every approved request gets an "is empty" comment. |
| empty | `JIRA_POSITION_FIELD`, `JIRA_DEPARTMENT_FIELD`, `JIRA_OFFICE_FIELD`, `JIRA_WEEKLY_HOURS_FIELD`, `JIRA_SUPERVISOR_FIELD`, `JIRA_LEGAL_ENTITY_FIELD` flow variables | Optional. Set the field IDs of the values you want sent; empty means the value is never sent. |
| `replace-with-jira-field-id` | `JIRA_PERSONIO_PERSON_ID_FIELD` flow variable | Required. Replace with the ID of the text field from setup step 1. |
| empty | `DEPARTMENT_ID_MAP`, `OFFICE_ID_MAP`, `LEGAL_ENTITY_ID_MAP` flow variables | Fill in `Jira value=Personio ID` pairs, see [Field Mappings](#field-mappings). |
| `https://api.personio.de/v2` | `PERSONIO_API_URL` flow variable | Leave as is. |
| `true` | `JIRA_COMMENTS_INTERNAL` flow variable | `true` for internal notes, `false` to show results to the requester. |
| Empty client ID and secret | `oauth2-personio` credential | Paste the Personio API client ID and client secret. |

No Jira custom field ID appears anywhere in the flow JSON; every one is a flow variable.

## Expected Success Result

- A new person with one employment exists in Personio, with the start date and every optional value the request supplied.
- The request's `Personio Employee ID` field holds the new ID, for example `3003`.
- An internal comment on the request reads: `Created Personio employee Ava Stone (ava.stone@example.com), Personio employee ID 3003, starting 2026-11-01.`

## Common Errors

| Comment or symptom | Cause | Fix |
| --- | --- | --- |
| `… is empty (Jira field replace-with-jira-field-id)` | The field flow variables were not configured. | Set the field IDs. |
| `Department "Ops" has no entry in DEPARTMENT_ID_MAP` | The Jira value is not in the map. | Add the pair, then move the request out of and back into `Approved`. |
| `Personio already has an employee with the email …` | The email is already used, for example a rehire or a resubmitted request. | Nothing was created. The request is linked to the existing employee. Handle rehires in Personio. |
| `the supervisor … was not found in Personio` | The supervisor's Jira email is not a Personio email. | Correct the supervisor. Also check that Jira exposes user emails, see [Limitations](#limitations). |
| `Personio rejected the new employee …: Bad request - …` | A Personio validation error: an unknown department, office or legal entity ID, a missing attribute that the Personio company has made mandatory, or an email already in use. | Read the detail, correct the request or the mapping, and retry. |
| `iHub could not search Personio …: Access Denied` | The credential lacks `personio:persons:read`. | Grant the permission and re-authorise the iHub credential. |
| No comment at all, error in the flow log | The token request failed (wrong client ID or secret, revoked credential), or Personio was unreachable. | Check the credential. See the engine note in [Limitations](#limitations). |
| Write-back fails with `400` from Jira | `JIRA_PERSONIO_PERSON_ID_FIELD` is wrong, or the field is not on the work item's edit screen. | Fix the field ID or the screen. The employee was created; the email guard stops a retry from duplicating it. |

## How Duplicate Processing Is Prevented

Jira redelivers webhooks, and an approved request keeps receiving `issue_updated` events as people comment and edit. Four guards, from cheapest to strongest:

1. **Transition only.** The `approvalTransition` scripted variable is `true` only when the event's changelog contains a `status` change whose new value is `APPROVED_STATUS_NAME`. Editing an approved request, commenting on it, or iHub writing the employee ID back all produce changelogs without that entry.
2. **Still approved.** Every Personio branch hangs below a fresh `GET` of the request, and only continues if the request is still in the approved status now. A redelivery that arrives after the request has moved on does nothing.
3. **Already linked.** A request whose `Personio Employee ID` field holds a value is ignored.
4. **Email lookup.** Before creating, the flow searches Personio for the email. If the write-back failed or the event was processed twice before the write-back landed, the second run finds the first employee and links to it instead of creating another. Personio also enforces email uniqueness itself.

The one remaining window is two deliveries of the same transition processed **concurrently**, both passing guard 4 before either create completes. Personio's email uniqueness should reject the second create, which then shows up as a failure comment rather than a duplicate employee.

## Common Lookup Pattern

The three Jira -> Personio templates each find employees with a `GET /v2/persons` action, and the field-reading scripted variables share one prelude. A shared helper flow was considered and rejected: no template in this repository invokes another flow and waits for its answer. Calling another flow's webhook URL is fire-and-forget, so its result cannot steer the calling flow. The lookup is a single action, so it is duplicated instead, with identical naming and conditions in each template.

## Limitations

- **One-way, create only.** Later changes to the request are not synced to Personio. Use `personio-employee-update-from-jira` for changes after the employee exists. Nothing in Personio is written back to Jira except the employee ID.
- **Failure comments assume iHub runs an action's children after a `4xx` response.** Every error branch is a child action that reads the failed call's problem body from `$.response.data.errors`. The monday and Linear templates read error bodies the same way, but they get `200` responses. No template in this repository proves the `4xx` case. If iHub stops a branch on a non-2xx status, Personio validation errors appear in the flow log only, and the request carries no comment. Verify this on your first test with a deliberately invalid request.
- **Office IDs are an assumption.** The employment's `office` takes an ID, but the v2 docs do not say which endpoint lists them. `GET /v2/workplaces` is the likely source. Verify one ID against an existing employee (`GET /v2/persons/{id}/employments`) before filling in `OFFICE_ID_MAP`.
- **Only the department org unit is sent.** Teams are not mapped. Adding one means a second `org_units` entry with `"type": "team"` and its own map.
- **User emails may be hidden.** A user picker's email is only in the webhook if the user's Atlassian profile makes it visible to apps. If supervisor emails arrive empty, use a short text field holding the email instead.
- **The `Customer Request Type` value shape is assumed.** The `requestTypeMatches` script reads `requestType.name` from the field value. If request types never match, check the field in the flow log and set `REQUEST_TYPE_NAME` empty as a workaround.
- **Only Personio's built-in employment fields are mapped.** Custom attributes (`custom_attributes: [{"id": "…", "value": "…"}]` on the person) are not. Adding one means another field flow variable, a scripted variable, and an entry in the create body.
- **Weekly hours must be a plain number.** A comma decimal (`37,5`) is accepted and converted; anything else is reported as invalid.
- **Personio v2 documents no numeric rate limit for persons.** The Personio actions carry the repository's standard `ratelimit` block, with three retries on reads and none on the create, so a timeout never creates twice.
- **Loops across templates.** See below.

### Loops Across Templates

The `personio` template's `Personio - Employee Onboarding` flow creates a JSM onboarding request when Personio reports a new employee. If Personio is configured to send that event for employees this template creates, every approved onboarding request produces a second, Personio-originated onboarding request. That is not an infinite loop. The second request only reaches this flow if someone approves it, and then the email lookup finds the employee and links to it instead of creating another. It is still duplicate work for the service desk. Do not route the Personio new-employee event for API-created employees back into the same JSM project, or point the two templates at different request types.

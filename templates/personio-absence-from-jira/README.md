# Jira Service Management - Create Personio Absence After Approval

This template installs one iHub flow that records an approved Jira Service Management leave request as an absence period in Personio.

- `Jira Service Management - Create Personio Absence After Approval`: Jira -> Personio, triggered by Jira webhooks.

The integration is **one-way**. Employees request leave in the JSM portal and the approval happens in Jira. Only after approval completes does iHub create the absence in Personio, which is the system of record from then on. Changes and cancellations in either system are not synchronised. See [Limitations](#limitations).

Related templates: `personio-employee-from-jira` and `personio-employee-update-from-jira` use the same credential, lookup and mapping conventions.

## What The Template Does

In execution order:

1. Listens for Jira `avi:jira:updated:issue` events.
2. Ignores everything except the update that **moves** a request of the configured project, work item type and (optionally) JSM request type **into** the approved status. Nothing is sent to Personio while a request is open, waiting for approval or declined.
3. Ignores requests that already hold a Personio absence period ID.
4. Re-reads the request from Jira, so a late redelivery of an event for a request that has since moved on is ignored.
5. Validates the request: the employee, a leave type with an entry in `ABSENCE_TYPE_ID_MAP`, start and end dates that are real dates, an end date that is not before the start date, and a one-day leave that does not both start and end at midday. Any problem stops the flow with one internal comment listing all of them.
6. Finds the employee in Personio, by the **Personio employee ID** on the request if one is configured and present, otherwise by **email**, which defaults to the **reporter's** email. Not found, more than one match, or a Personio error each stop the flow with a comment.
7. Creates the absence period in Personio with the mapped absence type, the dates and half-day flags, and a comment that names the Jira request. By default the absence skips Personio's own approval, because it has already been approved in Jira.
8. On success, writes the Personio absence period ID into the configured Jira field and comments with the result. On failure, for example an overlap with an existing absence, comments with Personio's reason.

## Example Jira Workflow

```
Open -> Waiting for approval -> Approved -> Done
                 |
                 +-> Declined
```

A `Request leave` request type in the HR project, with a leave type select (`Vacation`, `Sick leave`, `Parental leave`, …), a start date, an end date, two optional half-day checkboxes and an optional reason. A JSM approval step with the employee's manager as approver transitions the request to `Approved`. The flow runs on that transition, and only on that transition.

To retry after correcting a request, move it out of `Approved` and into it again.

## Required Jira Fields

| Data | Flow variable | Required | Jira field type | Personio field |
| --- | --- | --- | --- | --- |
| Personio employee ID | `JIRA_PERSONIO_PERSON_ID_FIELD` | No | Short text | Lookup key |
| Employee email | `JIRA_EMPLOYEE_EMAIL_FIELD` | Yes, unless the ID is always set | Default `reporter`; or a user picker or short text | Lookup key |
| Leave type | `JIRA_ABSENCE_TYPE_FIELD` | Yes | Select | `absence_type.id`, via `ABSENCE_TYPE_ID_MAP` |
| First day | `JIRA_START_DATE_FIELD` | Yes | Date picker | `starts_from.date_time` |
| Last day | `JIRA_END_DATE_FIELD` | Yes | Date picker | `ends_at.date_time` |
| Half day on first day | `JIRA_HALF_DAY_START_FIELD` | No | Checkbox or select | `starts_from.type` |
| Half day on last day | `JIRA_HALF_DAY_END_FIELD` | No | Checkbox or select | `ends_at.type` |
| Reason / comment | `JIRA_ABSENCE_COMMENT_FIELD` | No | Short or paragraph text | `comment` |
| Personio absence period ID | `JIRA_PERSONIO_ABSENCE_ID_FIELD` | Yes | Short text | Written by the flow |

Every field is referenced by its **field ID** in a flow variable, so the flow JSON names no custom field. Find a field's ID under `Settings` -> `Work items` -> `Custom fields`, or with `GET /rest/api/3/field`.

Values are normalised before use: a select option becomes its value, a user becomes their email address, a checkbox becomes its first ticked option, and a date-time becomes its date.

Half-day flags count as set for any value except empty, `No`, `False`, `0` or `None`. A single checkbox option `Yes`, or a `Yes`/`No` select, both work.

## Field Mappings

### Absence Types

Personio identifies absence types by UUID. `ABSENCE_TYPE_ID_MAP` maps the leave type values users pick in Jira to those UUIDs:

```
ABSENCE_TYPE_ID_MAP = Vacation=137a2048-41e9-4d49-9201-e6a032521495;Sick leave=0b1c…;Parental leave=7d3e…
```

- Pairs are `Jira value=Personio absence type ID`, separated by `;`. Matching is case-insensitive.
- A leave type **without an entry stops the flow** with the comment `Leave type "…" has no entry in ABSENCE_TYPE_ID_MAP`. This is the "absence type not mapped" case: nothing is sent to Personio.
- An **empty** map sends the Jira value unchanged as the ID, for a field that already holds Personio UUIDs.
- Labels cannot contain `;` or `=`.

List the IDs with the same credential:

```
GET https://api.personio.de/v2/absence-types
```

Each entry has `id`, `name`, `category` (for example `PAID_VACATION`, `SICK_LEAVE`) and `unit` (`DAY` or `HOUR`). Map only `DAY` types; see [Limitations](#limitations).

### Dates And Half Days

| Jira | Personio |
| --- | --- |
| First day `2026-12-21`, no half day | `"starts_from": { "date_time": "2026-12-21T00:00:00", "type": "FIRST_HALF" }` (full first day) |
| First day `2026-12-21`, half day | `"starts_from": { "date_time": "2026-12-21T00:00:00", "type": "SECOND_HALF" }` (starts at midday) |
| Last day `2026-12-23`, no half day | `"ends_at": { "date_time": "2026-12-23T00:00:00", "type": "SECOND_HALF" }` (full last day) |
| Last day `2026-12-23`, half day | `"ends_at": { "date_time": "2026-12-23T00:00:00", "type": "FIRST_HALF" }` (ends at midday) |

For day-based absence types Personio requires both `date_time` values at midnight, local time, with no timezone offset. Half days are expressed through `type`. For a one-day half-day leave, set the first and last day to the same date and tick only one of the two flags: the start flag gives an afternoon off, the end flag a morning off.

## Personio API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Confirm Leave Request Is Approved | `GET` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}?fields=status` (Jira) |
| Comment Invalid Leave Request on Jira | `POST` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/comment` (Jira) |
| Find Personio Employee | `GET` | `{{_flow.PERSONIO_API_URL}}/persons?id=…` or `?email=…`, `&limit=2` |
| Comment Personio Lookup Failure on Jira | `POST` | Jira comment |
| Comment Personio Employee Not Found on Jira | `POST` | Jira comment |
| Comment Ambiguous Personio Match on Jira | `POST` | Jira comment |
| Create Personio Absence Period | `POST` | `{{_flow.PERSONIO_API_URL}}/absence-periods?skip_approval={{_flow.PERSONIO_SKIP_APPROVAL}}` |
| Store Personio Absence ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` (Jira) |
| Comment Personio Absence Created on Jira | `POST` | Jira comment |
| Comment Personio Absence Failure on Jira | `POST` | Jira comment |

Things about the Personio API that shape the flow:

- **This uses `POST /v2/absence-periods`**, which answers `201` with `{"id": "fd19f19d-…"}`. The flow stores `$.response.data.id` in `personioAbsenceId`. The v1 `POST /v1/company/time-offs` endpoint still exists and is not marked deprecated, but it belongs to the v1 auth model that Personio is tightening, and v2 is where absence features are developed.
- **`skip_approval`.** By default Personio puts an API-created absence through the absence type's own approval workflow, leaving it `PENDING`. Approval has already happened in Jira, so the template sends `skip_approval=true` and the absence is recorded as approved. Set `PERSONIO_SKIP_APPROVAL` to `false` if Personio's approval must still apply, for example to keep Personio's substitute rules in force.
- **Overlaps are rejected with `409`**: `{"errors": [{"title": "Overlapping absence period", "detail": "Absence period overlaps with at least one existing period"}]}`. Other documented rejections are `409` "Substitute is invalid" or "Requesting absence while substituting", and `422` "Time frame end is before start", "Substitute required" or "Substitute not allowed". Each lands in the failure comment as title and detail.
- **The comment sent to Personio** is `Approved in Jira HR-7: <reason>`, cut to Personio's 2000-character limit, so an absence can always be traced back to its request.
- **Every Personio call sends `X-Personio-App-ID: IHUB_JIRA_INTEGRATION`**, the identification header Personio asks integrations to send.

## Body Format And Fidelity

The reason is copied as plain text. Every Jira value passes through a scripted variable that JSON-escapes it, so quotes, line breaks and backslashes survive. If the reason field is a paragraph (rich text) field, only its first text is read; use a plain short or paragraph text field.

Values returned by Personio (error title and detail) are emitted in the Jira comment as separate ADF text nodes through `{{{toJSON …}}}`, so a quote in a Personio message cannot break the comment.

## Credential

The manifest defines one credential: `oauth2-personio`, of type `OAUTH2` with the `client_credentials` grant against `https://api.personio.de/v2/auth/token`, sending parameters as `x-www-form-urlencoded`. This is the same shape `jamf-asset-import` uses, and what Personio's token endpoint requires.

1. In Personio, create API credentials under `Settings` -> `Integrations` -> `API credentials`, or `Marketplace` -> `Connected integrations` -> `Create custom integration`, depending on your Personio UI version.
2. Grant these permissions:

   | Permission | Scope string | Used by |
   | --- | --- | --- |
   | Read persons | `personio:persons:read` | `Find Personio Employee` |
   | Write absences | Not named in Personio's docs; `personio:absences:write` by the naming pattern | `Create Personio Absence Period` |
   | Read absences (setup only) | `personio:absences:read` | Listing absence type IDs with `GET /v2/absence-types` |

   Personio's reference page for creating absence periods does not state its scope. Grant the write permission for absences in the credential UI, and confirm with a test request.
3. In iHub, paste the **client ID** and **client secret** into the `oauth2-personio` credential, with no prefix. iHub fetches the `papi-…` bearer token, which is valid for 24 hours, and sends it as `Authorization: Bearer …`.

Changing a credential's scopes in Personio invalidates its tokens; re-authorise the iHub credential afterwards.

## Jira-side Setup

1. Create a single-line text field `Personio Absence ID` and add it to the leave request's work item screen. Put its ID in `JIRA_PERSONIO_ABSENCE_ID_FIELD`.
2. Build the leave request type with the leave type, start date, end date, and optional half-day and reason fields. Put each field's ID in its flow variable.
3. Make the leave type options match the labels in `ABSENCE_TYPE_ID_MAP`.
4. Decide how the employee is identified. The default `JIRA_EMPLOYEE_EMAIL_FIELD = reporter` assumes employees raise their own requests and that Jira exposes their email to apps. For requests raised on someone's behalf, point it at a user picker or email field instead.
5. Add an approval step that transitions to `APPROVED_STATUS_NAME`.
6. Set `REQUEST_TYPE_NAME` and `JIRA_REQUEST_TYPE_FIELD`. Leaving the request type empty matches every request of `REQUEST_ISSUE_TYPE` in the project, so only do that when no other request type there shares the work item type and approved status.
7. Enable the flow and approve a test request. Turn `logLevel` up while testing and back to `OFF` afterwards; absence data such as sick leave is sensitive.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `HR` | `SPACE` flow variable | Replace with the JSM project key. |
| `Service Request` | `REQUEST_ISSUE_TYPE` flow variable | Replace with the work item type of the leave request. |
| `Request leave` | `REQUEST_TYPE_NAME` flow variable | Replace with the JSM request type name, or empty it (see setup step 6). |
| `replace-with-request-type-field-id` | `JIRA_REQUEST_TYPE_FIELD` flow variable | Replace with the `Customer Request Type` field ID, often `customfield_10010`. |
| `Approved` | `APPROVED_STATUS_NAME` flow variable | Replace if your approved status is named differently. |
| empty | `JIRA_PERSONIO_PERSON_ID_FIELD` flow variable | Optional. A field holding the Personio employee ID, used before email. |
| `reporter` | `JIRA_EMPLOYEE_EMAIL_FIELD` flow variable | Keep for self-service requests, or replace with a field ID (setup step 4). |
| `replace-with-jira-field-id` | `JIRA_ABSENCE_TYPE_FIELD`, `JIRA_START_DATE_FIELD`, `JIRA_END_DATE_FIELD` flow variables | Required. Replace with the field IDs. |
| empty | `JIRA_HALF_DAY_START_FIELD`, `JIRA_HALF_DAY_END_FIELD`, `JIRA_ABSENCE_COMMENT_FIELD` flow variables | Optional field IDs. |
| empty | `ABSENCE_TYPE_ID_MAP` flow variable | Fill in `Jira leave type=Personio absence type ID` pairs. |
| `replace-with-jira-field-id` | `JIRA_PERSONIO_ABSENCE_ID_FIELD` flow variable | Required. Replace with the ID of the text field from setup step 1. |
| `true` | `PERSONIO_SKIP_APPROVAL` flow variable | `true` records approved absences; `false` also requires approval in Personio. |
| `https://api.personio.de/v2` | `PERSONIO_API_URL` flow variable | Leave as is. |
| `true` | `JIRA_COMMENTS_INTERNAL` flow variable | `true` for internal notes; `false` lets the employee see the confirmation in the portal. |
| Empty client ID and secret | `oauth2-personio` credential | Paste the Personio API client ID and secret. |

No Jira custom field ID appears anywhere in the flow JSON.

## Expected Success Result

- An approved absence period exists in Personio for the employee, with the mapped type, the dates and half days, and the comment `Approved in Jira HR-7: …`. It is `PENDING` instead if `PERSONIO_SKIP_APPROVAL` is `false`.
- The request's `Personio Absence ID` field holds the period's ID.
- An internal comment on the request reads: `Recorded in Personio: Vacation from 2026-12-21 from midday to 2026-12-23, for Personio employee 3003. Personio absence period ID fd19f19d-…. Personio approval skipped: true.`

## Common Errors

| Comment or symptom | Cause | Fix |
| --- | --- | --- |
| `Leave type "…" has no entry in ABSENCE_TYPE_ID_MAP` | The leave type is not mapped. | Add the pair, then move the request out of and back into `Approved`. |
| `End date … is before start date …` | Dates entered the wrong way round. | Correct and retry. |
| `Neither a Personio employee ID nor an employee email is set (Jira field reporter)` | Jira hides the reporter's email from apps, or the field is empty. | Use a dedicated employee email field. |
| `no Personio employee matches email …` | The Jira email differs from the Personio email. | Align the emails, or store the Personio employee ID on the request. |
| `Personio rejected …: Overlapping absence period - …` | The employee already has an absence in that period, possibly this same leave from an earlier run. | Check Personio. If the absence is there, put its ID in `Personio Absence ID` by hand. |
| `Personio rejected …: Substitute required` | The absence type requires a substitute in Personio. | Set `PERSONIO_SKIP_APPROVAL` to `false`, or record this type in Personio directly. |
| `Personio rejected …: Access Denied` or `403` | The credential lacks the absences write permission. | Grant it and re-authorise. |
| `Personio rejected …: Bad request - …` | An unknown absence type ID, or an hourly type that needs times. | Fix the map; map only day-based types. |
| No comment, error in the flow log | The token request failed, or Personio was unreachable. | Check the credential. See the engine note in [Limitations](#limitations). |

## How Duplicate Processing Is Prevented

Booking the same leave twice is the failure that matters most here, so there are four guards:

1. **Transition only.** The `approvalTransition` scripted variable is `true` only when the changelog contains a `status` change into `APPROVED_STATUS_NAME`. Comments and edits on an approved request, and the flow's own write-back, never re-run it.
2. **Still approved.** All Personio work hangs below a fresh `GET` of the request and continues only if it is still approved.
3. **Already recorded.** A request whose `Personio Absence ID` field holds a value is ignored.
4. **Personio overlap check.** If the same transition is processed twice before the write-back lands, or the write-back failed, Personio rejects the second absence with `409 Overlapping absence period`. The request gets a failure comment that says so, not a second absence.

## Common Lookup Pattern

The employee lookup (`personLookupQuery`: Personio ID first, email second, with not-found, ambiguous and error branches) is identical in `personio-employee-update-from-jira`. A shared helper flow was considered and rejected: no template in this repository invokes another flow and waits for its answer, and a webhook call to another flow is fire-and-forget.

## Limitations

- **One-way, create only.** Cancelling, shortening or extending the leave in Jira after approval does nothing in Personio, and absences entered or changed in Personio are not reflected in Jira. Cancellation would need a second flow on the transition to a `Cancelled` status that calls `DELETE /v2/absence-periods/{id}` with the ID this template stores.
- **Day-based absence types only.** Hourly types (`unit: HOUR`) need start and end times with minute precision; the template sends midnight dates. Do not map hourly types.
- **Timezone.** Personio expects local times with no offset and does not document which timezone it applies when the request has none; presumably the employee's. Since only midnight dates are sent, this only matters for employees whose Personio timezone differs from their office's calendar.
- **No working-day or balance check in Jira.** Personio computes the effective days and balance. The Jira request does not show them; the employee sees them in Personio.
- **Substitutes are not sent.** If an absence type requires a substitute, Personio rejects the request with `422 Substitute required`.
- **The absences write scope is unverified.** Personio's docs do not name the scope for creating absence periods; see [Credential](#credential).
- **Failure comments assume iHub runs an action's children after a `4xx` response.** Each error branch is a child action that reads the failed call's problem body from `$.response.data.errors`. No template in this repository proves the `4xx`/`409` case. If iHub stops a branch on a non-2xx status, the overlap and validation errors appear in the flow log only. The duplicate guard still holds, because Personio rejects the overlap either way. Verify on your first test by approving the same leave twice.
- **Reporter email visibility.** Jira only includes a user's email in the webhook if their Atlassian profile allows it. With the default `reporter` mapping, hidden emails produce the "neither … is set" comment.
- **The `Customer Request Type` value shape is assumed** to carry `requestType.name`. If requests never match, check the field in the flow log, or empty `REQUEST_TYPE_NAME`.
- **No numeric rate limit is documented for Personio v2 absences.** The lookup retries up to three times. The create never retries, so a timeout cannot book twice.

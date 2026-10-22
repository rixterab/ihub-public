# Jira Service Management - Update Employee in Personio

This template installs one iHub flow that applies an approved Jira Service Management HR change request to an existing employee in Personio: a new department, office, position, weekly working hours, supervisor or legal entity.

- `Jira Service Management - Update Employee in Personio`: Jira -> Personio, triggered by Jira webhooks.

The integration is **one-way**. Jira is where the change is requested and approved; Personio is the system of record afterwards. Nothing is read back from Personio except to confirm the change in a comment. See [Limitations](#limitations).

Related templates: `personio-employee-from-jira` creates the employee in the first place, `personio-absence-from-jira` records approved leave, and the `personio` template creates JSM mover requests *from* Personio events. See [Loops Across Templates](#loops-across-templates).

## What The Template Does

In execution order:

1. Listens for Jira `avi:jira:updated:issue` events.
2. Ignores everything except the update that **moves** a request of the configured project, work item type and (optionally) JSM request type **into** the approved status.
3. Re-reads the request from Jira, so a late redelivery of an event for a request that has since moved on is ignored.
4. Validates the request: an employee identifier must be present, at least one attribute must be changed, weekly hours must be a number, and mapped values must have an entry in their map. Any problem stops the flow with one internal comment listing all of them.
5. Finds the employee in Personio, by the **Personio employee ID** on the request if it holds one, otherwise by **email**. Not found, more than one match, or a Personio error each stop the flow with a comment.
6. If the employee was found by email and the request has a Personio employee ID field, writes the ID into it, so a re-run looks the employee up by ID.
7. Reads the employee's most recent employment. Department, office, position, hours, supervisor and legal entity live on the employment, not on the person.
8. Resolves the new supervisor's email to a Personio person ID, if a new supervisor was requested. If that person is not in Personio, the flow stops and **nothing** is changed.
9. Sends a `PATCH` to the employment containing **only** the attributes the request supplies. Anything left empty in Jira is not sent and keeps its current Personio value.
10. On success, reads the employment back and comments with what Personio now holds. On failure, comments with Personio's error title and detail.

## Example Jira Workflow

```
Open -> Waiting for approval -> Approved -> Done
                 |
                 +-> Declined
```

A typical request type, `Change employee details`, has the employee (a user picker or an email field) plus optional fields for the new department, office, position, weekly hours, supervisor and legal entity. The requester fills in only what changes. A JSM approval step, usually the employee's manager and HR, transitions the request to `Approved`, and the flow runs on that transition.

To retry after correcting a request, move it out of `Approved` and into it again.

## Required Jira Fields

| Data | Flow variable | Required | Jira field type | Personio field |
| --- | --- | --- | --- | --- |
| Personio employee ID | `JIRA_PERSONIO_PERSON_ID_FIELD` | No, preferred | Short text | Lookup key; filled in by the flow when empty |
| Employee email | `JIRA_EMPLOYEE_EMAIL_FIELD` | Yes, unless the ID is always set | User picker or short text | Lookup key |
| New position | `JIRA_POSITION_FIELD` | No | Short text or select | `position.title` |
| New department | `JIRA_DEPARTMENT_FIELD` | No | Select or short text | `org_units[]` with `type: department`, via `DEPARTMENT_ID_MAP` |
| New office | `JIRA_OFFICE_FIELD` | No | Select or short text | `office.id`, via `OFFICE_ID_MAP` |
| New weekly working hours | `JIRA_WEEKLY_HOURS_FIELD` | No | Number | `weekly_working_hours` |
| New supervisor | `JIRA_SUPERVISOR_FIELD` | No | User picker, or short text holding an email | `supervisor.id`, resolved by email |
| New legal entity | `JIRA_LEGAL_ENTITY_FIELD` | No | Select or short text | `legal_entity.id`, via `LEGAL_ENTITY_ID_MAP` |

At least one of the six change fields must hold a value on each request.

Every field is referenced by its **field ID** in a flow variable, so the flow JSON names no custom field. Find a field's ID under `Settings` -> `Work items` -> `Custom fields`, or with `GET /rest/api/3/field`. Built-in fields work by their system ID, for example `reporter`.

Values are normalised before use: a select option becomes its value, a user picker becomes the user's email address, a multi-value field becomes its first entry.

### Employee Lookup Priority

1. **Personio employee ID** from `JIRA_PERSONIO_PERSON_ID_FIELD`: `GET /v2/persons?id=…`.
2. **Email** from `JIRA_EMPLOYEE_EMAIL_FIELD`: `GET /v2/persons?email=…`.

The lookup asks for two results, so a second match is detected and reported rather than guessed at. Both keys are unique in Personio, so an ambiguous match points at bad data.

A third, configurable identifier (an employee number or another custom attribute) is **not supported**. `GET /v2/persons` filters only on `id`, `email`, `first_name`, `last_name`, `preferred_name`, `status` and timestamps, and of those only `id` and `email` are unique. Matching on a custom attribute would mean paging through every employee, which an iHub action cannot do and then filter.

## Field Mappings

Personio v2 references departments, offices, legal entities and supervisors **by ID**. Three flow variables translate the names users pick in Jira:

```
DEPARTMENT_ID_MAP   = Engineering=943;Sales=944
OFFICE_ID_MAP       = Stockholm=231;Berlin=232
LEGAL_ENTITY_ID_MAP = Rixter AB=465
```

- Pairs are `Jira value=Personio ID`, separated by `;`. Matching is case-insensitive.
- A value with no entry stops the flow with a comment.
- An **empty** map sends the Jira value unchanged as the ID, for fields that already hold Personio IDs.
- Labels cannot contain `;` or `=`.

Read the IDs with `GET /v2/org-units?type=department`, `GET /v2/workplaces` (see [Limitations](#limitations)) and `GET /v2/legal-entities`.

The supervisor is resolved by email with `GET /v2/persons?email=…`; no map is needed.

## Personio API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Confirm Change Request Is Approved | `GET` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}?fields=status` (Jira) |
| Comment Invalid Change Request on Jira | `POST` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/comment` (Jira) |
| Find Personio Employee | `GET` | `{{_flow.PERSONIO_API_URL}}/persons?id=…` or `?email=…`, `&limit=2` |
| Comment Personio Lookup Failure on Jira | `POST` | Jira comment |
| Comment Personio Employee Not Found on Jira | `POST` | Jira comment |
| Comment Ambiguous Personio Match on Jira | `POST` | Jira comment |
| Store Personio Employee ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` (Jira) |
| Find Current Personio Employment | `GET` | `{{_flow.PERSONIO_API_URL}}/persons/{person_id}/employments?limit=1` |
| Comment No Personio Employment on Jira | `POST` | Jira comment |
| Find Personio Supervisor | `GET` | `{{_flow.PERSONIO_API_URL}}/persons?email=…&limit=1` |
| Comment Personio Supervisor Not Found on Jira | `POST` | Jira comment |
| Update Personio Employment | `PATCH` | `{{_flow.PERSONIO_API_URL}}/persons/{person_id}/employments/{employment_id}` |
| Comment Personio Update Failure on Jira | `POST` | Jira comment |
| Read Back Personio Employment | `GET` | `{{_flow.PERSONIO_API_URL}}/persons/{person_id}/employments/{employment_id}` |
| Comment Personio Employee Updated on Jira | `POST` | Jira comment |

Things about the Personio API that shape the flow:

- **These attributes belong to the employment, not the person.** `PATCH /v2/persons/{id}` only changes person fields (name, email, gender, language, custom attributes). Department, office, position, hours, supervisor and legal entity are changed with `PATCH /v2/persons/{person_id}/employments/{employment_id}`. The flow reads the employee's employments, which Personio returns most recent first, and changes the first.
- **The update answers `204 No Content`.** Success is detected as "no `errors` array in the response", and the flow then reads the employment back with `GET …/employments/{id}`. The confirmation comment reports the values Personio holds after the change, not the values the flow sent.
- **Errors are `application/problem+json`**: `{"errors": [{"title": "…", "detail": "…"}]}`. The first error's title and detail go into the failure comment.
- **There is no effective date.** The v2 employment update applies immediately. A change approved today for next month is applied today. See [Limitations](#limitations).
- **The supervisor lookup always runs.** iHub skips an action's children when its condition is not met, so an optional lookup cannot sit in the middle of a chain. Without a supervisor change, the query is unfiltered (`GET /v2/persons?limit=1` returns any one person) and the `PATCH` body leaves the supervisor out.
- **Every Personio call sends `X-Personio-App-ID: IHUB_JIRA_INTEGRATION`**, the identification header Personio asks integrations to send.

## Body Format And Fidelity

The `PATCH` body is built from the `employmentChanges` scripted variable, which emits only the members the request supplies, with `JSON.stringify`, so values are escaped correctly. The supervisor is appended by the action body itself, because its ID is only known after the supervisor lookup:

```
{ {{{employmentChanges}}}{{#if changeSupervisorEmail}}{{#if employmentChanges}},{{/if}}"supervisor": { "id": "{{_flow.personioSupervisorId}}" }{{/if}} }
```

A request that changes the position and the supervisor sends exactly:

```json
{ "position": { "title": "Team Lead" }, "supervisor": { "id": "4223" } }
```

Jira values in comments are JSON-escaped by their scripted variables. Values that come back **from Personio** (names, position title, error text) are emitted as separate ADF text nodes through `{{{toJSON …}}}`, so a quote in them cannot break the comment.

## Credential

The manifest defines one credential: `oauth2-personio`, of type `OAUTH2` with the `client_credentials` grant against `https://api.personio.de/v2/auth/token`, sending parameters as `x-www-form-urlencoded`. This is the same shape `jamf-asset-import` uses, and what Personio requires.

1. In Personio, create API credentials under `Settings` -> `Integrations` -> `API credentials`, or under `Marketplace` -> `Connected integrations` -> `Create custom integration`, depending on your Personio UI version.
2. Grant these permissions:

   | Permission | Scope string | Used by |
   | --- | --- | --- |
   | Read persons | `personio:persons:read` | `Find Personio Employee`, `Find Current Personio Employment`, `Find Personio Supervisor`, `Read Back Personio Employment` |
   | Write persons | `personio:persons:write` | `Update Personio Employment` (Personio has no separate employment scope) |

3. In iHub, paste the **client ID** and **client secret** into the `oauth2-personio` credential, exactly as Personio shows them and with no prefix. iHub fetches the `papi-…` bearer token itself. It is valid for 24 hours.

Changing a credential's scopes in Personio invalidates its tokens; re-authorise the iHub credential afterwards. The same Personio credential works for all three Jira -> Personio templates if it holds the union of their scopes.

## Jira-side Setup

1. Optionally create a single-line text field `Personio Employee ID` and put its ID in `JIRA_PERSONIO_PERSON_ID_FIELD`. If it is set, the flow fills it in on the first run and looks the employee up by ID from then on. Leave the variable empty to always look up by email.
2. Build the change request type with an employee field and the change fields you need. Put each field's ID in its flow variable, and leave the ones you do not use empty.
3. Add an approval step that transitions to `APPROVED_STATUS_NAME`.
4. Fill in the mapping variables, or leave them empty if the Jira fields hold Personio IDs.
5. Set `REQUEST_TYPE_NAME` and `JIRA_REQUEST_TYPE_FIELD`. Leaving the request type empty matches every request of `REQUEST_ISSUE_TYPE` in the project, so only do that when the project has no other request type that shares the work item type and approved status.
6. Enable the flow and approve a test request. Turn `logLevel` up while testing and back to `OFF` afterwards, because the flow log holds employee data.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `HR` | `SPACE` flow variable | Replace with the JSM project key. |
| `Service Request` | `REQUEST_ISSUE_TYPE` flow variable | Replace with the work item type of the change request. |
| `Change employee details` | `REQUEST_TYPE_NAME` flow variable | Replace with the JSM request type name, or empty it (see setup step 5). |
| `replace-with-request-type-field-id` | `JIRA_REQUEST_TYPE_FIELD` flow variable | Replace with the `Customer Request Type` field ID, often `customfield_10010`. |
| `Approved` | `APPROVED_STATUS_NAME` flow variable | Replace if your approved status is named differently. |
| empty | `JIRA_PERSONIO_PERSON_ID_FIELD` flow variable | Optional, recommended. Set to the ID of the `Personio Employee ID` text field. |
| `replace-with-jira-field-id` | `JIRA_EMPLOYEE_EMAIL_FIELD` flow variable | Required. Replace with the ID of the field identifying the employee. |
| empty | `JIRA_POSITION_FIELD`, `JIRA_DEPARTMENT_FIELD`, `JIRA_OFFICE_FIELD`, `JIRA_WEEKLY_HOURS_FIELD`, `JIRA_SUPERVISOR_FIELD`, `JIRA_LEGAL_ENTITY_FIELD` flow variables | Set the field IDs for the attributes your request type can change. |
| empty | `DEPARTMENT_ID_MAP`, `OFFICE_ID_MAP`, `LEGAL_ENTITY_ID_MAP` flow variables | Fill in `Jira value=Personio ID` pairs. |
| `https://api.personio.de/v2` | `PERSONIO_API_URL` flow variable | Leave as is. |
| `true` | `JIRA_COMMENTS_INTERNAL` flow variable | `true` for internal notes, `false` to show results to the requester. |
| Empty client ID and secret | `oauth2-personio` credential | Paste the Personio API client ID and secret. |

No Jira custom field ID appears anywhere in the flow JSON.

## Expected Success Result

- The employee's current employment in Personio holds the new values. Every attribute the request did not mention is unchanged.
- If `JIRA_PERSONIO_PERSON_ID_FIELD` is set and was empty, it now holds the employee's Personio ID.
- An internal comment on the request reads:

  ```
  Updated Personio employee 3003 Avery Stone. Requested: position: Team Lead, supervisor: boss@example.com.
  Personio now holds: weekly working hours 40; office ID 231; supervisor ID 4223; position Team Lead
  ```

## Common Errors

| Comment or symptom | Cause | Fix |
| --- | --- | --- |
| `The request does not change any employment attribute` | All change fields are empty, or their flow variables are not set. | Fill in a change, or configure the field variables. |
| `no employee matches email …` | The email is not a Personio email, or the Personio ID on the request is wrong. | Correct the request. Check that Jira exposes the user's email, see [Limitations](#limitations). |
| `more than one employee matches …` | Duplicate data in Personio. | Put the Personio employee ID on the request and retry. |
| `employee … has no employment in Personio` | The person has no employment record. | Create the employment in Personio. |
| `the new supervisor … was not found in Personio` | The supervisor's email is not in Personio. Nothing was changed. | Correct the supervisor and retry. |
| `Department "…" has no entry in DEPARTMENT_ID_MAP` | Missing mapping. | Add the pair and retry. |
| `Personio rejected the change …: Bad request - …` | An unknown org unit, office or legal entity ID, or an invalid value. | Fix the mapping or the value and retry. |
| `iHub could not search Personio …: Access Denied` | The credential lacks `personio:persons:read`. | Grant it and re-authorise. |
| `Personio now holds` shows the old value | Personio accepted the request but did not apply that attribute, for example a team-only org unit change. | Check the employee in Personio. The comment reports Personio's state on purpose. |
| No comment, error in the flow log | Token request failed, or Personio was unreachable. | Check the credential. See the engine note in [Limitations](#limitations). |

## How Duplicate Processing Is Prevented

An update is naturally idempotent: applying the same `PATCH` twice leaves the same state. The guards therefore stop repeated *work* and repeated *comments*, not corruption:

1. **Transition only.** The `approvalTransition` scripted variable is `true` only when the changelog contains a `status` change into `APPROVED_STATUS_NAME`. Edits and comments on an already approved request, and the flow's own employee ID write-back, do not re-run it.
2. **Still approved.** All Personio work hangs below a fresh `GET` of the request and continues only if it is still approved.
3. **Same values.** If the same transition is delivered twice, the second `PATCH` writes the same values and the request gets a second, identical confirmation comment.

The one case where a re-run changes something is someone editing the change fields and then re-approving. That is a new decision, and applying it is intended.

## Common Lookup Pattern

The three Jira -> Personio templates share the same lookup: one `GET /v2/persons` action fed by the `personLookupQuery` scripted variable (ID first, email second), with identical not-found, ambiguous and error branches. A shared helper flow was considered and rejected: no template in this repository invokes another flow and waits for its answer, and calling another flow's webhook URL is fire-and-forget, so the result could not steer this flow.

## Limitations

- **One-way.** Personio changes are never written back to Jira. Changes made directly in Personio are not reflected on the request.
- **Changes apply immediately.** The Personio v2 employment update has no effective date. For a future-dated change, approve the request on or near the effective date. A scheduled variant would need a date field and a scheduled flow that searches for approved requests due today.
- **The department replaces the employment's org units.** Personio documents `org_units` as "all Org Units … to be associated with the Employment" and does not say whether a `PATCH` merges or replaces. Assume a department change can **remove the employee's team**. Verify on a test employee before production use. If it does, add the team to the change, or extend `employmentChanges` to keep the current team from `Find Current Personio Employment`.
- **The most recent employment is changed.** An employee with several concurrent employments only has the first one returned by Personio updated.
- **Office IDs are an assumption.** The v2 docs do not say which endpoint lists office IDs; `GET /v2/workplaces` is the likely source. Check one against an existing employee before filling in `OFFICE_ID_MAP`.
- **Person attributes are not changed.** Name, email and custom attributes go through `PATCH /v2/persons/{id}`, which this flow does not call. Adding one means another action after `Find Personio Employee`, with its own success and failure comments.
- **Failure comments assume iHub runs an action's children after a `4xx` response.** Every error branch is a child action that reads the failed call's problem body from `$.response.data.errors`. No template in this repository proves the `4xx` case. If iHub stops a branch on a non-2xx status, failures appear in the flow log only. Verify on your first test with a deliberately invalid department ID.
- **Success detection relies on an empty `204` body comparing as "no errors".** This matches how the Linear and monday templates check for an absent `errors` array. If a Personio failure ever arrived without a problem body, the read-back comment would still show Personio's real state, so the request would not claim a change that did not happen.
- **User emails may be hidden** from apps by Atlassian profile visibility. If emails arrive empty, use short text fields holding the email.
- **The `Customer Request Type` value shape is assumed** to carry `requestType.name`. If requests never match, check the field in the flow log, or empty `REQUEST_TYPE_NAME`.
- **No numeric rate limit is documented for Personio v2 persons.** The Personio actions carry the standard `ratelimit` block with three retries; every call in this flow is safe to repeat.

### Loops Across Templates

The `personio` template's `Personio - Employee Mover` flow creates a JSM mover request when Personio reports a department, manager, role or location change. If Personio sends that event for changes this template makes, every approved change request produces a Personio-originated mover request. This is not an infinite loop, because the mover request needs its own approval before it could reach this flow, and would then apply the same values again. It is still duplicate work. Do not route Personio mover events for API-made changes into the same JSM project, or use different request types for the two directions.

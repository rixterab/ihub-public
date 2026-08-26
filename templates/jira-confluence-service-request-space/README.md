# JSM Service Request Approved to Confluence Space

This template installs one iHub flow that creates a dedicated Confluence space when a Jira Service Management service request is approved, then links that space back to the request.

Trigger: `JSM Service Request Approved` -> `Fetch Request` -> `Create Confluence Space` -> `Link Space Back to Jira`.

The flow listens to the Atlassian webhook trigger of the instance iHub is installed in (`ATLASSIAN_WEBHOOK_TRIGGER_TYPE`). Confluence is reached through the `CONFLUENCE_URL` flow variable with a separate `BASIC_AUTH` credential, since the Confluence Cloud site is not always the same tenant as the Jira Service Management site iHub is installed in.

## What The Template Does

1. **Fetch Approved Service Request**: on a Jira issue updated event, checks that the issue type and status match the customer's definition of an approved service request, and that no space has been created for it yet. Fetches the summary, rendered description, reporter and project key.
2. **Create Confluence Space**: builds a Confluence-safe space key from the issue key (letters and numbers only, optionally prefixed) and creates a new space named `<issue key> - <summary>` with a short description referencing the request and requester.
3. **Store Space Link on Service Request**: writes the new space URL into a Jira custom field on the request. This is also the guard that stops a second space from being created on the next update.
4. **Add Remote Link to Confluence Space**: creates a Jira remote link (`/remotelink`) pointing at the space, so it shows up as a linked item on the request.
5. **Comment Confluence Space Link on Service Request**: posts a comment on the request with the space link, so the requester and agents see it without opening the custom field.

## Jira And Confluence API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Fetch service request | `GET` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}?fields=...&expand=renderedFields` |
| Create Confluence space | `POST` | `{{_flow.CONFLUENCE_URL}}/rest/api/space` |
| Store space link on request | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` |
| Add remote link | `POST` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/remotelink` |
| Comment on request | `POST` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/comment` |

Calls to Jira use the built-in `ATLASSIAN_TOKEN` credential reference and need no manual configuration. Calls to Confluence use the manifest's `basic-auth-confluence` credential, since the Confluence site may be a different Atlassian tenant than the one iHub is installed in. If Jira and Confluence are on the same site, the same Atlassian account email and API token can be reused for both.

## Building The Space Key

Confluence space keys only allow letters and numbers. The `Create Confluence Space` action derives one with a scripted variable: it takes the `SPACE_KEY_PREFIX` flow variable, appends the issue key, uppercases the result and strips everything that is not `A-Z0-9`, so `PROJ-123` with prefix `SR` becomes `SRPROJ123`. Space keys must also be unique across the Confluence site; since Jira issue keys are unique, a collision can only happen if a space with that derived key already exists for an unrelated reason, in which case the create call fails and the flow does not retry with a different key.

## Description Body

The incident-style HTML rendering used in the sibling PIR template is not used here: the space description is Confluence's plain-text description field (`description.plain`), not a page body, so it is built directly from Jira field values rather than `renderedFields`.

## Required Customer Configuration

Create a Jira custom field of type single-line text (or URL) on the service request to hold the Confluence space link. The template uses `customfield_10161`; every customer must replace this with the field id from their Jira instance, in the flow variable list, the `Fetch Approved Service Request` action condition, and the `Store Space Link on Service Request` action body.

Hard-coded values that must be reviewed before enabling the flow:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `Service Request` | `SERVICE_REQUEST_ISSUE_TYPE` flow variable **and** `Fetch Approved Service Request` action condition | Replace with the issue type name used for service requests in this instance. |
| `Approved` | `APPROVED_STATUS_NAME` flow variable **and** `Fetch Approved Service Request` action condition | Replace with the status name that marks a service request as approved in this instance's workflow. If approval is tracked with the native JSM approvals feature rather than a status transition, change the condition to check the approval decision field instead. |
| `customfield_10161` | Flow variable, action condition and the write-back action body | Replace with the custom field id used to store the space link, for example `customfield_12345`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | `IHUB_ACCOUNT_ID` flow variable **and** the `Fetch Approved Service Request` action condition | Replace with the `accountId` of the iHub integration user in this Jira instance, in both places. This is the loop guard for the space link write-back. |
| `https://your-company.atlassian.net/wiki` | `CONFLUENCE_URL` flow variable | Replace with the base URL of the Confluence Cloud site, for example `https://acme.atlassian.net/wiki`. |
| `SR` | `SPACE_KEY_PREFIX` flow variable | Replace with the prefix to use when building space keys, or set to an empty string to use the bare issue key. |
| `integration@your-company.com` | `manifest.json` credential definition | Replace with the Atlassian account email used for the Confluence API token, before importing or by editing the credential after import. |

Condition values in iHub are compared literally and are not templated, which is why the issue type, status and account id above are hard coded in the action condition in addition to being flow variables. The flow variables are the documented reference value; the conditions are what actually gates the flow, so both must be kept in sync.

To find an `accountId`, call `GET /rest/api/3/myself` in the Jira instance while authenticated as the integration user.

## Credentials

The manifest defines one credential: `basic-auth-confluence`, of type `BASIC_AUTH`.

- **Username**: the Atlassian account email of the integration user used to call Confluence.
- **Password**: an Atlassian API token created by that user at `https://id.atlassian.com/manage-profile/security/api-tokens`.

The Confluence integration user needs the site-level "Create space(s)" global permission. The Jira integration user (via `ATLASSIAN_TOKEN`) needs permission to view the service request, edit the space link custom field, add comments and create remote links.

## Loop Prevention

Writing the space link back onto the request fires another `jira:issue_updated` webhook. Two guards stop that from creating a second space:

1. The action condition only proceeds when the space link field is empty. Once step 3 writes the link, every later event for the same request fails this condition.
2. The action condition also skips events authored by the iHub integration user (`IHUB_ACCOUNT_ID`), which covers the write-back, remote link creation and comment.

As with the sibling PIR template, there is a narrow race if two qualifying updates land before the first space finishes creating. This is treated as acceptable for v1, not fully closed.

## Known Limitations

Out of scope for v1:

- **Space permissions.** The new space is created with default Confluence permissions (space admins get the site default). Granting the requester, their team, or a specific group access is not automated and needs the Confluence space permissions API added separately.
- **Space template / starter pages.** The space is created empty aside from its description. Seeding it with a homepage, page tree or blueprint is not included.
- **Re-approval.** If a service request is re-approved after being reopened, the link field guard prevents a second space from being created. The existing space is not touched.
- **Space key collisions.** If the derived space key is already taken by an unrelated space, the create call fails and the flow does not retry with a different key or clean up.

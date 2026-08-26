# JSM Incident to Confluence Post-Incident Report

This template installs one iHub flow that turns a resolved Jira Service Management major incident into a Confluence post-incident report (PIR), then links that report back to the incident.

Trigger: `JSM Major Incident Resolved` -> `Fetch Incident + Comments` -> `Create Confluence PIR` -> `Link Page Back to Jira`.

The flow listens to the Atlassian webhook trigger of the instance iHub is installed in (`ATLASSIAN_WEBHOOK_TRIGGER_TYPE`). Confluence is reached through the `CONFLUENCE_URL` flow variable with a separate `BASIC_AUTH` credential, since Confluence Cloud sites are not always the same tenant as the Jira Service Management site iHub is installed in.

## What The Template Does

1. **Fetch Resolved Incident**: on a Jira issue updated event, checks that the issue type, priority and status match the customer's definition of a resolved major incident, and that the issue is not already linked to a report. Fetches the issue with its description rendered as HTML (`expand=renderedFields`) plus priority, assignee, reporter, created and resolved timestamps.
2. **Create Confluence PIR Page**: creates a page in the configured Confluence space titled `PIR: <issue key> - <summary>`, with a metadata table (priority, reporter, assignee, created, resolved, a link back to the Jira issue) followed by the rendered incident description.
3. **Fetch Incident Comments**: fetches every comment on the incident with `expand=renderedBody`, then iterates over them.
4. **Add Comment to Confluence Page**: for each incident comment, adds a matching comment on the new Confluence page (author and timestamp, followed by the rendered comment body), so the incident timeline is mirrored under the report.
5. **Store PIR Link on Incident**: writes the new Confluence page URL into a Jira custom field on the incident. This is also the guard that stops a second report from being created on the next update to the same incident.
6. **Add Remote Link to PIR Page**: creates a Jira remote link (`/remotelink`) pointing at the Confluence page, so it shows up as a linked item on the incident alongside any pages linked by hand.

## Jira And Confluence API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Fetch incident | `GET` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}?fields=...&expand=renderedFields` |
| Create PIR page | `POST` | `{{_flow.CONFLUENCE_URL}}/rest/api/content` |
| Fetch incident comments | `GET` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/comment?expand=renderedBody&orderBy=created` |
| Add comment to PIR page | `POST` | `{{_flow.CONFLUENCE_URL}}/rest/api/content/{{_flow.pirPageId}}/child/comment` |
| Store PIR link on incident | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` |
| Add remote link | `POST` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}/remotelink` |

Calls to Jira use the built-in `ATLASSIAN_TOKEN` credential reference and need no manual configuration. Calls to Confluence use the manifest's `basic-auth-confluence` credential, since the Confluence site may be a different Atlassian tenant than the one iHub is installed in. If Jira and Confluence are on the same site, the same Atlassian account email and API token can be reused for both.

## Description And Comment Bodies

The incident description and comments arrive from Jira as ADF (Atlassian Document Format). Rather than converting ADF to HTML on the fly, the template asks Jira to do the rendering by requesting `expand=renderedFields` on the issue fetch and `expand=renderedBody` on the comment fetch. Jira returns ready-to-use HTML fragments (`renderedFields.description`, and `renderedBody` per comment), which are inserted directly into the Confluence storage-format body. This keeps formatting, links and lists intact without a custom ADF-to-storage-format converter, at the cost of any Jira-specific rendering (smart links, embedded Jira macros) not carrying over.

## Required Customer Configuration

Create a Jira custom field of type single-line text (or URL) on the incident to hold the PIR page link. The template uses `customfield_10160`; every customer must replace this with the field id from their Jira instance, in both the flow variable list and the `Fetch Resolved Incident` action condition and the `Store PIR Link on Incident` action body.

Hard-coded values that must be reviewed before enabling the flow:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `Incident` | `MAJOR_INCIDENT_ISSUE_TYPE` flow variable **and** `Fetch Resolved Incident` action condition | Replace with the issue type name used for major incidents in this instance. |
| `Highest` | `MAJOR_INCIDENT_PRIORITY` flow variable **and** `Fetch Resolved Incident` action condition | Replace with the priority name used to mark a major incident. |
| `Resolved` | `RESOLVED_STATUS_NAME` flow variable **and** `Fetch Resolved Incident` action condition | Replace with the status name that marks an incident as resolved in this instance's workflow. |
| `customfield_10160` | Flow variable, action condition and both write-back action bodies | Replace with the custom field id used to store the PIR link, for example `customfield_12345`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | `IHUB_ACCOUNT_ID` flow variable **and** the `Fetch Resolved Incident` action condition | Replace with the `accountId` of the iHub integration user in this Jira instance, in both places. This is the loop guard for the link write-back. |
| `https://your-company.atlassian.net/wiki` | `CONFLUENCE_URL` flow variable | Replace with the base URL of the Confluence Cloud site, for example `https://acme.atlassian.net/wiki`. |
| `PIR` | `CONFLUENCE_SPACE_KEY` flow variable | Replace with the key of the Confluence space that should hold post-incident reports. |
| (empty) | `CONFLUENCE_PARENT_PAGE_ID` flow variable | Optionally set to a Confluence page id to create every report as a child of that page instead of at the space root. |
| `integration@your-company.com` | `manifest.json` credential definition | Replace with the Atlassian account email used for the Confluence API token, before importing or by editing the credential after import. |

Condition values in iHub are compared literally and are not templated, which is why the issue type, priority, status and account id above are hard coded in the action condition in addition to being flow variables. The flow variables are the documented reference value; the conditions are what actually gates the flow, so both must be kept in sync.

To find an `accountId`, call `GET /rest/api/3/myself` in the Jira instance while authenticated as the integration user.

## Credentials

The manifest defines one credential: `basic-auth-confluence`, of type `BASIC_AUTH`.

- **Username**: the Atlassian account email of the integration user used to call Confluence.
- **Password**: an Atlassian API token created by that user at `https://id.atlassian.com/manage-profile/security/api-tokens`.

The Confluence integration user needs permission to add pages and comments in the configured space. The Jira integration user (via `ATLASSIAN_TOKEN`) needs permission to view the incident, edit the PIR link custom field, and create remote links.

## Loop Prevention

Writing the PIR link back onto the incident fires another `jira:issue_updated` webhook. Two guards stop that from creating a second report:

1. The action condition only proceeds when the PIR link field is empty. Once step 5 writes the link, every later event for the same incident fails this condition.
2. The action condition also skips events authored by the iHub integration user (`IHUB_ACCOUNT_ID`), which covers the write-back itself and the remote link creation.

There is a narrow race if two qualifying updates land before the first report finishes creating; the second could pass the empty-field check before the first write completes. This is treated the same way other sync templates in this repository treat it: acceptable for v1, not fully closed.

## Known Limitations

Out of scope for v1:

- **Attachments.** Only the incident description and comments are copied; attachments on the incident are not uploaded to the Confluence page.
- **Re-opened incidents.** If an incident is reopened and resolved again, the link field guard prevents a second report from being created. The existing PIR page is not updated.
- **Custom incident templates.** The report layout (summary table plus description plus comments) is fixed. Sections for root cause, impact or action items are not templated in and would need to be added to the `Create Confluence PIR Page` body manually.
- **Jira smart rendering.** `renderedFields`/`renderedBody` HTML does not preserve Jira-specific elements such as smart links or issue status lozenges; they render as plain text or are dropped.

# Snyk Findings to Jira

Polls Snyk for open findings on a schedule and raises a Jira issue for each one that does not already have an open issue.

- `Snyk Findings to Jira`: Snyk -> Jira, on `0 6 * * *`, disabled by default.

This is the repository's first security template. It is a poll rather than a webhook because Snyk's webhooks fire on project snapshots rather than on individual issues, which makes them a poor fit for per-finding ticketing.

## What The Template Does

| Order | Action | Purpose |
| --- | --- | --- |
| 100 | Fetch Open Snyk Issues | `GET /rest/orgs/{org}/issues?status=open`, following `links.next` |
| 200 | Find Jira Issue for Snyk Finding | Iterates `$.payload.data` and searches Jira per finding |
| 300 | Create Jira Issue from Snyk Finding | Creates when the search came back empty |

Action 200 carries the iterator, so 300 runs once per finding. Findings whose severity is not listed in `SNYK_SEVERITIES` are dropped at 200 and never reach 300.

Created issues are labelled `snyk` and `snyk-<severity>`, and carry the Snyk issue ID in a custom field, which is what makes the run idempotent: a finding that already has an open Jira issue is skipped on every later run.

## Snyk API

| Action | Method | Endpoint |
| --- | --- | --- |
| Fetch Open Snyk Issues | `GET` | `{{_flow.SNYK_API_URL}}/rest/orgs/{{_flow.SNYK_ORG_ID}}/issues?version={{_flow.SNYK_API_VERSION}}&limit=100&status=open` |

Two things about the Snyk REST API shape the flow:

- **`version` is mandatory on every request.** Snyk pins its REST API by date. `SNYK_API_VERSION` holds it; bumping it may change the response shape, so re-check the field paths after a bump.
- **`links.next` is a relative path**, not an absolute URL. The pagination variable prefixes `SNYK_API_URL` when the value does not already start with `http`.

Regional API hosts: `https://api.snyk.io` (default), `https://api.eu.snyk.io`, `https://api.au.snyk.io`. The API host and the app host differ; both are flow variables.

## Credential

The manifest defines one credential: `custom-header-snyk-token`, of type `CUSTOM_HEADER` with a single `Authorization` header.

Create a service account under `Settings` -> `Service accounts` in the Snyk organization with the **Org Viewer** role, and paste the full header value including the prefix. Snyk uses `token`, not `Bearer`:

```
Authorization: token 12345678-1234-1234-1234-123456789012
```

## Required Customer Configuration

Create a Jira custom field of type single-line text to hold the Snyk issue ID. The template uses `customfield_10156` / `cf[10156]`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `SEC` | `SPACE` flow variable | Replace with the Jira project key for security findings. |
| `Bug` | `ISSUE_TYPE` flow variable | Replace with the issue type used for vulnerabilities. |
| `https://api.snyk.io` | `SNYK_API_URL` flow variable | Replace for the EU or AU region. |
| `https://app.snyk.io` | `SNYK_APP_URL` flow variable | Replace for the EU or AU region. |
| `00000000-…` | `SNYK_ORG_ID` flow variable | Replace with the Snyk organization UUID. |
| `your-org` | `SNYK_ORG_SLUG` flow variable | Replace with the Snyk organization slug used in web links. |
| `2024-10-15` | `SNYK_API_VERSION` flow variable | Bump only deliberately; re-verify the field paths afterwards. |
| `critical,high` | `SNYK_SEVERITIES` flow variable | Replace with the severities that should raise a Jira issue. |
| `customfield_10156` / `cf[10156]` | Flow JSON, two places | Replace with the customer's Jira custom field key and numeric ID. |
| `snyk.svg` | Template folder | Placeholder mark, not the Snyk logo. Replace with the official asset before publishing. |

Start with `critical` only. An organization enabling this against a mature codebase at `critical,high` can open hundreds of Jira issues on the first run; the flow has no cap.

## Limitations

- **One-way and create-only.** Fixing the vulnerability in Snyk does not close the Jira issue, and closing the Jira issue does not ignore the finding in Snyk. A second scheduled flow that searches Jira for `snyk` issues and checks each against the Snyk API would close that gap.
- **Org-scoped.** One Snyk organization per flow. Install once per org, or add a second fetch action.
- **No project grouping.** Every finding becomes its own Jira issue. Teams that prefer one issue per project per scan need a different shape.
- The Snyk link in the description is built from `coordinates[0]` and is best-effort; findings with an unusual coordinate shape get a link that does not resolve. The Snyk issue ID in the custom field is the reliable identifier.
- No first-run guard. See the note about `SNYK_SEVERITIES` above.

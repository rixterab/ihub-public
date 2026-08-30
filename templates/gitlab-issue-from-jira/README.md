# GitLab - Create Issue from Jira

This template installs one iHub flow that creates a GitLab issue whenever a work item is created in a configured Jira project.

- `GitLab - Create Issue from Jira`: Jira -> GitLab, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from GitLab into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in the `SPACE` flow variable.
- Ignores work items that already carry a GitLab issue IID, so a redelivered webhook never creates a second GitLab issue.
- Creates one GitLab issue in the project configured in `GITLAB_PROJECT_ID`, with the labels from `GITLAB_LABELS`.
- Writes the created GitLab issue IID back to a Jira custom field, which is what makes the previous step work.

The GitLab issue description carries a link back to the Jira work item, the Jira project key, issue type, priority and reporter, and the converted Jira description.

## GitLab API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Create GitLab Issue | `POST` | `{{_flow.GITLAB_API_URL}}/projects/{{_flow.GITLAB_PROJECT_ID}}/issues` |
| Store GitLab Issue IID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` |

The create action stores `$.response.data.iid` in `gitlabIssueIid` and `$.response.data.web_url` in `gitlabIssueUrl`. Store the **IID**, not the global `id`: every project-scoped GitLab issue endpoint takes the IID, and that is the value written back to Jira.

`GITLAB_PROJECT_ID` accepts either the numeric project ID or the URL-encoded `namespace/project` path, for example `acme%2Fbackend`.

## Body Format And Fidelity

Jira Cloud descriptions are ADF documents, not strings. The template passes them through `adfToHTML`, the same conversion the `github-issues-sync` template uses. GitLab renders inline HTML inside Markdown descriptions, so the result is readable, though it is HTML rather than native GitLab Markdown.

Free-text Jira values (summary, priority, reporter) are interpolated with triple braces so an `&` or an apostrophe is not turned into an HTML entity. As in every other template in this repository, a double quote inside a Jira summary can still break the surrounding JSON body.

## Credential

The manifest defines one credential: `custom-header-gitlab-token`, of type `CUSTOM_HEADER` with a single `PRIVATE-TOKEN` header.

Create a project or group access token in GitLab under `Settings` -> `Access tokens` with the `api` scope, and paste the raw token as the header value. No prefix:

```
PRIVATE-TOKEN: glpat-xxxxxxxxxxxxxxxxxxxx
```

A personal access token works too, but a project access token scoped to the target project is the better choice for a template.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the GitLab issue IID. The template uses `customfield_10153`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `GITLAB` | `SPACE` flow variable | Replace with the Jira project key that should create GitLab issues. |
| `https://gitlab.com/api/v4` | `GITLAB_API_URL` flow variable | Replace for self-managed GitLab, for example `https://gitlab.example.com/api/v4`. |
| `12345678` | `GITLAB_PROJECT_ID` flow variable | Replace with the GitLab project ID or URL-encoded `namespace/project` path. |
| `jira` | `GITLAB_LABELS` flow variable | Replace with the comma-separated labels to apply in GitLab. |
| `customfield_10153` | Flow JSON, two places | Replace with the customer's Jira custom field key. |
| Empty `PRIVATE-TOKEN` header | `custom-header-gitlab-token` credential | Set to a GitLab token with the `api` scope. |

### Places To Update The Jira Custom Field

- `Create GitLab Issue` condition: checks `$.issue.fields.customfield_10153` is empty before creating.
- `Store GitLab Issue IID on Jira Issue` body: writes the created IID into `customfield_10153`.

## Limitations

- One-way only. GitLab issue changes, comments and closures are not written back to Jira. Building the other direction means a second flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a GitLab project webhook, in the shape of `github-issues-sync`. The IID write-back in this template is the link the incoming flow would search on.
- Create only. Later Jira updates, comments and attachments are not synced to the GitLab issue.
- One Jira project and one GitLab project per flow. Install the template once per pair.
- The write-back is itself a Jira update, so it fires an `issue_updated` webhook. This flow only acts on `avi:jira:created:issue`, so it ignores it.

# Bitbucket - Create Issue from Jira

This template installs one iHub flow that creates a Bitbucket Cloud issue whenever a work item is created in a configured Jira project.

- `Bitbucket - Create Issue from Jira`: Jira -> Bitbucket, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from Bitbucket into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in the `SPACE` flow variable.
- Ignores work items that already carry a Bitbucket issue ID, so a redelivered webhook never creates a second issue.
- Creates one Bitbucket issue in the configured repository, with `kind: bug` and `priority: major`.
- Writes the created Bitbucket issue ID back to a Jira custom field, which is what makes the previous step work.

Issue tracking must be enabled on the Bitbucket repository under `Repository settings` -> `Issue tracker`. It is off by default, and the create call returns `404` when it is off.

## Bitbucket API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Bitbucket Issue | `POST` | `https://api.bitbucket.org/2.0/repositories/{{_flow.BITBUCKET_WORKSPACE}}/{{_flow.BITBUCKET_REPO_SLUG}}/issues` |
| Store Bitbucket Issue ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` |

The create action stores `$.response.data.id` in `bitbucketIssueId`.

## Body Format And Fidelity

Jira Cloud descriptions are ADF documents, not strings. The template passes them through `adfToHTML` and sets `content.markup` to `html`, so Bitbucket renders the converted description rather than showing markup as literal text.

Free-text Jira values (summary, priority, reporter) are interpolated with triple braces so an `&` or an apostrophe is not turned into an HTML entity. As in every other template in this repository, a double quote inside a Jira summary can still break the surrounding JSON body.

`kind` and `priority` are fixed values. Bitbucket accepts `bug`, `enhancement`, `proposal` and `task` for `kind`, and `trivial`, `minor`, `major`, `critical` and `blocker` for `priority`. Neither maps cleanly onto Jira issue types or priorities, so the template does not guess; map them in the body if your process needs it.

## Credential

The manifest defines one credential: `basic-auth-bitbucket`, of type `BASIC_AUTH`.

Create an app password in Bitbucket under `Personal settings` -> `App passwords` with the `Issues: Write` permission. Use your Bitbucket **username** (not the email address) as the username and the app password as the password. The placeholder in the manifest is only a placeholder.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Bitbucket issue ID. The template uses `customfield_10153`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `BITBUCKET` | `SPACE` flow variable | Replace with the Jira project key that should create Bitbucket issues. |
| `workspace-id` | `BITBUCKET_WORKSPACE` flow variable | Replace with the Bitbucket workspace slug. |
| `repository-slug` | `BITBUCKET_REPO_SLUG` flow variable | Replace with the repository slug. |
| `customfield_10153` | Flow JSON, two places | Replace with the customer's Jira custom field key. |
| `bitbucket-user@example.com` | `basic-auth-bitbucket` credential | Replace with the Bitbucket username and set the app password. |

### Places To Update The Jira Custom Field

- `Create Bitbucket Issue` condition: checks `$.issue.fields.customfield_10153` is empty before creating.
- `Store Bitbucket Issue ID on Jira Issue` body: writes the created ID into `customfield_10153`.

## Limitations

- One-way only. Bitbucket issue changes and comments are not written back to Jira. The other direction needs a second flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a Bitbucket repository webhook; the ID write-back here is the link it would search on.
- Create only. Later Jira updates, comments and attachments are not synced.
- Bitbucket Cloud only. Bitbucket Data Center has a different API surface.
- One Jira project and one repository per flow.

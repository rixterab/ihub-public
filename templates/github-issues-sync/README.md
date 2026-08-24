# Jira <-> GitHub Issues Sync

This template installs two iHub flows that keep Jira issues and GitHub issues linked in both directions.

- `GitHub Issues Outgoing Sync`: Jira -> GitHub, triggered by Jira webhooks.
- `GitHub Issues Incoming Sync`: GitHub -> Jira, triggered by GitHub issue webhooks calling an iHub custom webhook URL.

The sync stores the GitHub issue number in a Jira custom field. That field is the link key used by both flows to find the matching record. iHub is the master orchestration point: Jira and GitHub send webhook data to iHub, and iHub decides whether to create, update, comment on, or link attachment data between issues.

## What The Template Syncs

From Jira to GitHub:

- Creates a GitHub issue when a Jira issue is created and the Jira issue does not already have a GitHub issue number.
- Stores the created GitHub issue number back on the Jira issue.
- Updates the GitHub issue title and body when the Jira issue is updated.
- Adds Jira comments to the linked GitHub issue as issue comments.
- Adds Jira attachments to the linked GitHub issue as comments containing Jira attachment links.
- Stores synced GitHub comment IDs as Jira issue properties using keys like `ihub-<github-id>`.

From GitHub to Jira:

- Searches Jira for an issue whose GitHub issue number custom field matches the incoming GitHub issue number.
- Creates a Jira issue when GitHub sends an `issues` / `opened` webhook event and no matching Jira issue exists.
- Updates the linked Jira issue from GitHub issue update events.
- Adds GitHub issue comments to the matching Jira issue as comments.
- Supports a custom attachment payload with an attachment URL, downloads it, and uploads it to Jira.
- Stores incoming GitHub comment and attachment IDs as Jira issue properties using keys like `ihub-<github-id>` to avoid duplicates.

## GitHub API Calls

The outgoing flow uses GitHub REST API endpoints:

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Issue | `POST` | `{{_flow.GITHUB_API_URL}}/repos/{{_flow.GITHUB_OWNER}}/{{_flow.GITHUB_REPO}}/issues` |
| Update Issue | `PATCH` | `{{_flow.GITHUB_API_URL}}/repos/{{_flow.GITHUB_OWNER}}/{{_flow.GITHUB_REPO}}/issues/{{issue.fields.customfield_10153}}` |
| Add Comment | `POST` | `{{_flow.GITHUB_API_URL}}/repos/{{_flow.GITHUB_OWNER}}/{{_flow.GITHUB_REPO}}/issues/{{issue.fields.customfield_10153}}/comments` |
| Add Attachment Link Comment | `POST` | `{{_flow.GITHUB_API_URL}}/repos/{{_flow.GITHUB_OWNER}}/{{_flow.GITHUB_REPO}}/issues/{{issue.fields.customfield_10153}}/comments` |

GitHub authenticates with a personal access token or GitHub App installation token that can read and write issues in the target repository.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the GitHub issue number. The template currently uses `customfield_10153` / `cf[10153]`; every customer must replace this with the field ID from their Jira instance.

This template assumes one configured GitHub repository per instantiated template. GitHub issue numbers are only unique inside a repository.

Hard-coded values that must be reviewed before enabling the flows:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://api.github.com` | Outgoing flow variable `GITHUB_API_URL` | Keep for GitHub.com, or replace with the GitHub Enterprise Server API URL. |
| `2026-03-10` | Outgoing flow variable `GITHUB_API_VERSION` | Replace only if the customer standardizes on another GitHub REST API version. |
| `your-org` | Outgoing flow variable `GITHUB_OWNER` | Replace with the target repository owner or organization. |
| `your-repo` | Outgoing flow variable `GITHUB_REPO` | Replace with the target repository name. |
| `GH` | Incoming flow variable `SPACE` | Replace with the Jira project key where GitHub-created issues should create/find issues. |
| `Task` | Incoming flow variable `ISSUE_TYPE` | Replace if the target Jira project does not use `Task`. |
| `customfield_10153` | Both flow JSON files | Replace with the customer's Jira custom field key, for example `customfield_12345`. |
| `cf[10153]` | Incoming flow JQL search | Replace with the numeric custom field ID, for example `cf[12345]`. |
| `712020:6a00297e-29ca-4539-8eba-272c510f6e9d` | Outgoing comment and attachment conditions | Replace with the Jira account ID used by iHub/the integration user. |
| `ihub-sync-bot` | Incoming GitHub event conditions | Replace with the GitHub login used by the iHub GitHub token. This prevents circular issue and comment sync. |

## Places To Update The Jira Custom Field

Update all references to the GitHub issue number field, not only the create action.

In `github_issues_outgoing_sync.json`:

- `Create Issue in GitHub` condition: checks `$.issue.fields.customfield_10153` is empty before creating an issue.
- `Store GitHub Issue Number on Jira Issue` body: writes the created GitHub issue number back to `customfield_10153`.
- `Add Jira Comment to GitHub Issue` URL and condition: uses `issue.fields.customfield_10153`.
- `Update GitHub Issue from Jira Issue` URL, flow variable, and condition: uses `issue.fields.customfield_10153`.
- `Add Jira Attachment Link to GitHub Issue` URL and condition: uses `issue.fields.customfield_10153`.

In `github_issues_incoming_sync.json`:

- `Find Jira Issue by GitHub Issue Number` JQL: searches `cf[10153]` for the incoming GitHub issue number.
- `Create Jira Issue from GitHub Issue` body: writes the incoming GitHub issue number into `customfield_10153`.

## GitHub Webhook Setup

After the template is instantiated in iHub, copy the trigger URL from the `GitHub Issues Incoming Sync` custom webhook trigger.

In GitHub:

1. Go to the repository `Settings` -> `Webhooks`.
2. Add a webhook pointing to the iHub custom webhook URL.
3. Use `application/json` as the content type.
4. Select `Let me select individual events`.
5. Enable `Issues` and `Issue comments`.
6. Add a webhook secret if the iHub custom webhook is configured to validate one.

Native GitHub issue webhooks send payloads like these.

Issue opened or updated event:

```json
{
  "action": "opened",
  "issue": {
    "number": 42,
    "title": "Example issue",
    "body": "Issue body in Markdown",
    "state": "open",
    "html_url": "https://github.com/owner/repo/issues/42"
  },
  "sender": {
    "login": "octocat"
  }
}
```

Issue comment event:

```json
{
  "action": "created",
  "issue": {
    "number": 42,
    "title": "Example issue"
  },
  "comment": {
    "id": 123456789,
    "body": "Comment body in Markdown",
    "user": {
      "login": "octocat"
    }
  },
  "sender": {
    "login": "octocat"
  }
}
```

The incoming flow also accepts common custom variants such as top-level `githubIssueNumber`, `issueNumber`, `commentBody`, `commentId`, and nested `issue`, `comment`, and `attachment` objects. Pull request issue-comment payloads are ignored by the parser.

## Attachment Notes

GitHub Issues comments support Markdown links to uploaded files, but GitHub does not expose a public REST endpoint for uploading arbitrary binary files directly into an issue comment. Because of that, Jira -> GitHub attachment sync creates a GitHub issue comment with a link back to the Jira attachment.

GitHub -> Jira binary attachment sync is available only for custom webhook payloads that include an attachment ID and URL, for example:

```json
{
  "eventType": "issue.attachment.created",
  "issue": {
    "number": 42
  },
  "attachment": {
    "id": "github-attachment-id",
    "name": "example.pdf",
    "url": "https://..."
  }
}
```

For native GitHub issue comments, file references in Markdown are synced as comment text.

## Loop Prevention

The outgoing flow blocks Jira comments and attachments authored by the configured iHub integration user. Incoming GitHub events block the configured GitHub integration login and check Jira issue properties before creating comments or attachments. This prevents updates created by one side of the sync from being immediately sent back to the source side.

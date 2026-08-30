# GitHub Actions - Dispatch Workflow from Jira

This template installs one iHub flow that dispatches a GitHub Actions workflow when a work item of a configured type is created in a configured Jira project.

- `GitHub Actions - Dispatch Workflow from Jira`: Jira -> GitHub Actions, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from GitHub into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores work items outside the project in `SPACE` or of an issue type other than `TRIGGER_ISSUE_TYPE`.
- Calls the `workflow_dispatch` endpoint with `jira_issue_key`, `jira_issue_url` and `jira_summary` as inputs.

## GitHub API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Dispatch GitHub Actions Workflow | `POST` | `{{_flow.GITHUB_API_URL}}/repos/{{_flow.GITHUB_OWNER}}/{{_flow.GITHUB_REPO}}/actions/workflows/{{_flow.GITHUB_WORKFLOW}}/dispatches` |

**The workflow must already declare `workflow_dispatch` with matching inputs on the default branch**, or GitHub answers `422`. The `workflow_dispatch` trigger is only recognised once the workflow file exists on the repository's default branch, even when `GITHUB_REF` points elsewhere.

```yaml
name: Jira Triggered
on:
  workflow_dispatch:
    inputs:
      jira_issue_key:
        description: Jira work item key
        required: false
        type: string
      jira_issue_url:
        description: Link back to the Jira work item
        required: false
        type: string
      jira_summary:
        description: Jira work item summary
        required: false
        type: string
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Triggered by ${{ inputs.jira_issue_key }}"
```

**A successful dispatch answers `204 No Content`.** There is no run ID in the response, so the flow captures nothing; correlating the run back to Jira means reading `jira_issue_key` from inside the workflow.

`GITHUB_WORKFLOW` accepts either the file name (`jira-triggered.yml`) or the numeric workflow ID.

## Credential

The manifest defines one credential: `token-github-actions`, of type `TOKEN`, which sends `Authorization: Bearer <token>`.

A fine-grained personal access token needs **Actions: read and write** on the target repository. A classic token needs the `repo` scope (or `public_repo` for public repositories). Paste the token alone, without a prefix.

The summary is interpolated with triple braces so an apostrophe is not turned into an HTML entity; as elsewhere in this repository, a double quote inside a Jira summary can still break the JSON body.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `GHA` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `Task` | `TRIGGER_ISSUE_TYPE` flow variable | Replace with the Jira issue type that should dispatch a workflow. |
| `https://api.github.com` | `GITHUB_API_URL` flow variable | Replace for GitHub Enterprise Server. |
| `2026-03-10` | `GITHUB_API_VERSION` flow variable | Bump only deliberately. |
| `your-org` / `your-repo` | `GITHUB_OWNER` / `GITHUB_REPO` flow variables | Replace with the target repository. |
| `jira-triggered.yml` | `GITHUB_WORKFLOW` flow variable | Replace with the workflow file name or ID. |
| `main` | `GITHUB_REF` flow variable | Replace with the ref the workflow runs against. |
| Empty token | `token-github-actions` credential | Set to a PAT with Actions write access. |

## Limitations

- One-way. Workflow results are not written back to Jira. Reporting status needs a second flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a GitHub `workflow_run` webhook.
- No run ID. See the API section above.
- No deduplication. A redelivered create webhook dispatches a second run.
- `workflow_dispatch` inputs are capped at 10 and are all strings.
- One Jira project, one issue type and one workflow per flow.

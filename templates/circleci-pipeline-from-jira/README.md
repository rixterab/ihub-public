# CircleCI - Trigger Pipeline from Jira

This template installs one iHub flow that triggers a CircleCI pipeline when a work item is created in a configured Jira project.

- `CircleCI - Trigger Pipeline from Jira`: Jira -> CircleCI, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from CircleCI into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in `SPACE`.
- Triggers one pipeline on the configured project and branch, passing the Jira key, URL and summary as pipeline parameters.

## CircleCI API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Trigger CircleCI Pipeline | `POST` | `https://circleci.com/api/v2/project/{{_flow.CIRCLECI_VCS}}/{{_flow.CIRCLECI_ORG}}/{{_flow.CIRCLECI_REPO}}/pipeline` |

The action stores `$.response.data.id` in `circleciPipelineId`.

The three pipeline parameters sent are `jira_issue_key`, `jira_issue_url` and `jira_summary`. **CircleCI rejects the request with `400` if the `.circleci/config.yml` on the target branch does not declare matching parameters.** Add them before enabling the flow:

```yaml
version: 2.1
parameters:
  jira_issue_key:
    type: string
    default: ""
  jira_issue_url:
    type: string
    default: ""
  jira_summary:
    type: string
    default: ""
```

Pipeline parameters are only readable from `config.yml` itself, not from job steps, so pass them into a job through an environment variable if a script needs them.

## Credential

The manifest defines one credential: `custom-header-circleci-token`, of type `CUSTOM_HEADER` with a single `Circle-Token` header.

Create a personal API token in CircleCI under `User settings` -> `Personal API tokens` and paste the raw token as the header value. No prefix:

```
Circle-Token: your-circleci-token
```

The token must belong to a user with write access to the project. CircleCI has no project-scoped API token for this endpoint, so use a dedicated machine user rather than a personal account.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `CIRCLECI` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `gh` | `CIRCLECI_VCS` flow variable | Replace with `gh` (GitHub) or `bb` (Bitbucket). |
| `organization` | `CIRCLECI_ORG` flow variable | Replace with the VCS organization or workspace. |
| `repository` | `CIRCLECI_REPO` flow variable | Replace with the repository name. |
| `main` | `CIRCLECI_BRANCH` flow variable | Replace with the branch to run. |
| Empty `Circle-Token` header | `custom-header-circleci-token` credential | Set to a CircleCI API token. |

## Limitations

- One-way only. The pipeline result is not written back to Jira. Reporting status needs a second flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a CircleCI webhook.
- The pipeline ID is captured but not stored on the Jira work item, so there is no link to follow later.
- No deduplication. A redelivered create webhook triggers a second pipeline.
- Triggering on every created work item is rarely what a customer wants. Add a condition on issue type or label so only the intended work items trigger a build.
- `gh`/`bb` project slugs are the legacy CircleCI addressing scheme. Projects on GitHub App integration use `circleci/<project-id>` triggers instead; check which your project uses.

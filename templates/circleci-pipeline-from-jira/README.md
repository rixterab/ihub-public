# CircleCI - Trigger Pipeline from Jira

Trigger a CircleCI pipeline when a Jira work item is created in the configured project.

The template installs one flow:

- `CircleCI - Trigger Pipeline from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `CIRCLECI` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `gh` | `CIRCLECI_VCS` flow variable | Replace with `gh` or `bb`. |
| `organization` | `CIRCLECI_ORG` flow variable | Replace with the VCS org or workspace. |
| `repository` | `CIRCLECI_REPO` flow variable | Replace with the repository name. |
| `main` | `CIRCLECI_BRANCH` flow variable | Replace with the branch to run. |
| `CircleCI Token` | `manifest.json` credential | Add a token allowed to trigger pipelines. |

The flow passes Jira metadata as pipeline parameters. The CircleCI config must define matching parameters before they can be used by jobs.

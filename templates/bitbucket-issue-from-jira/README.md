# Bitbucket - Create Issue from Jira

Create a Bitbucket Cloud issue when a Jira work item is created in the configured project.

The template installs one flow:

- `Bitbucket - Create Issue from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `BITBUCKET` | `SPACE` flow variable | Replace with the Jira project key that should create Bitbucket issues. |
| `workspace-id` | `BITBUCKET_WORKSPACE` flow variable | Replace with the Bitbucket workspace slug. |
| `repository-slug` | `BITBUCKET_REPO_SLUG` flow variable | Replace with the repository slug. |
| `Bitbucket Cloud` | `manifest.json` credential | Use a Bitbucket username and app password with issue write access. |

Bitbucket issue tracking must be enabled on the repository.

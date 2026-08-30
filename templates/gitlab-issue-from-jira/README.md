# GitLab - Create Issue from Jira

Create a GitLab issue when a Jira work item is created in the configured project.

The template installs one flow:

- `GitLab - Create Issue from Jira`

## Setup

| Value in template | Where | Customer action |
| --- | --- | --- |
| `GITLAB` | `SPACE` flow variable | Replace with the Jira project key that should create GitLab issues. |
| `https://gitlab.com/api/v4` | `GITLAB_API_URL` flow variable | Replace if using self-managed GitLab. |
| `12345678` | `GITLAB_PROJECT_ID` flow variable | Replace with the GitLab project ID or URL-encoded namespace/project path. |
| `jira` | `GITLAB_LABELS` flow variable | Replace with labels to add to created GitLab issues. |
| `GitLab Token` | `manifest.json` credential | Add a token that can create issues in the target project. |

The body maps Jira summary, description, priority, reporter, and issue URL into the GitLab issue description. Add a write-back step if you want to store the GitLab issue URL on the Jira issue.

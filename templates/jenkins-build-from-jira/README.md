# Jenkins - Trigger Job from Jira

This template installs one iHub flow that triggers a parameterised Jenkins job when a work item of a configured type is created in a configured Jira project.

- `Jenkins - Trigger Job from Jira`: Jira -> Jenkins, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from Jenkins into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores work items outside the project in `SPACE` or of an issue type other than `TRIGGER_ISSUE_TYPE`.
- Calls `buildWithParameters` on the configured job, passing `JIRA_ISSUE_KEY`, `JIRA_ISSUE_URL` and `JIRA_ISSUE_TYPE`.
- Captures the queue item URL from the `Location` response header into `jenkinsQueueUrl`.

## Jenkins API Call

| Action | Method | Endpoint |
| --- | --- | --- |
| Trigger Jenkins Job | `POST` | `{{_flow.JENKINS_URL}}/job/{{_flow.JENKINS_JOB}}/buildWithParameters?JIRA_ISSUE_KEY=…` |

Three things about Jenkins shape this action:

- **The job must be parameterised.** Add `JIRA_ISSUE_KEY`, `JIRA_ISSUE_URL` and `JIRA_ISSUE_TYPE` as String parameters, or Jenkins ignores the values. A non-parameterised job needs `/build` instead of `/buildWithParameters`.
- **Parameters go in the query string**, and the request sends `Content-Type: application/x-www-form-urlencoded` with an empty body. Jenkins does not read a JSON body here.
- **`buildWithParameters` answers `201 Created` with no body.** The build ID is not returned; the `Location` header points at a queue item, and that queue item has to be polled to learn the build number. The flow captures the header and stops there.

For a job inside a folder, `JENKINS_JOB` must include the folder segments in Jenkins' own form, for example `my-folder/job/my-job`.

## Credential

The manifest defines one credential: `basic-auth-jenkins`, of type `BASIC_AUTH`.

In Jenkins go to the user's `Configure` page and create an **API token**. Use the Jenkins username as the username and the API token as the password — not the account password. Modern Jenkins exempts API-token requests from CSRF, which is why this flow does not fetch a crumb first; if your instance still demands one, a preceding `GET /crumbIssuer/api/json` action is needed.

Use a dedicated Jenkins service account with `Job/Build` on the target job rather than a person's account.

## Required Customer Configuration

| Value in template | Where | Customer action |
| --- | --- | --- |
| `JENKINS` | `SPACE` flow variable | Replace with the Jira project key to monitor. |
| `Task` | `TRIGGER_ISSUE_TYPE` flow variable | Replace with the Jira issue type that should trigger a build. |
| `https://jenkins.example.com` | `JENKINS_URL` flow variable | Replace with the Jenkins base URL, no trailing slash. |
| `my-job` | `JENKINS_JOB` flow variable | Replace with the job name, including `folder/job/name` for folders. |
| `jenkins-user` | `basic-auth-jenkins` credential | Replace with the Jenkins username and set the API token as the password. |
| `jenkins.svg` | Template folder | Placeholder mark, not the Jenkins logo. Replace with the official asset before publishing. |

## Limitations

- One-way. Build results are not written back to Jira. Reporting status needs a second flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a Jenkins notification plugin.
- No build number. See the API section above.
- No deduplication. A redelivered create webhook triggers a second build.
- Jenkins must be reachable from iHub. An instance behind a VPN with no ingress will not work; consider the `useStaticIP` option and an allow-list.
- One Jira project, one issue type and one job per flow.

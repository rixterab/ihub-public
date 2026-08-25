# Okta

Import Okta resources into Jira Assets.

The `Okta` flow is a library of `WEB_REQUEST_ACTION_TYPE` actions for reading resources out of an Okta org via the Okta Management API. It is disabled by default and has no scheduled trigger configured (`ATLASSIAN_WEBHOOK_TRIGGER_TYPE` placeholder) — actions are intended to be run individually (e.g. via Test Flow) or wired into an import flow per customer, rather than run as-is.

## What It Reads

| Action | Method | Endpoint |
| --- | --- | --- |
| Get Okta apps | `GET` | `{{_flow.OKTA_ADMIN_URL}}/apps` (auto-paginated via the `Link` response header) |
| Get Users in App | `GET` | `{{_flow.OKTA_URL}}/apps/{{_flow.SAMPLE_APP_ID}}/users` (disabled by default) |
| Get roles | `GET` | `{{_flow.OKTA_ADMIN_URL}}/iam/roles` |
| Get Policies | `GET` | `{{_flow.OKTA_ADMIN_URL}}/policies?type=OKTA_SIGN_ON` |
| Get Users | `GET` | `{{_flow.OKTA_ADMIN_URL}}/users` |
| Get Policies PASSWORD | `GET` | `{{_flow.OKTA_ADMIN_URL}}/policies?type=PASSWORD` |
| Get Policies MFA_ENROLL | `GET` | `{{_flow.OKTA_ADMIN_URL}}/policies?type=MFA_ENROLL` |
| Get Policies IDP_DISCOVERY | `GET` | `{{_flow.OKTA_ADMIN_URL}}/policies?type=IDP_DISCOVERY` |
| Get Groups | `GET` | `{{_flow.OKTA_ADMIN_URL}}/groups` |
| Get Groups for app | `GET` | `{{_flow.OKTA_URL}}/apps/{{_flow.SAMPLE_APP_ID}}/groups` |
| Get Types | `GET` | `{{_flow.OKTA_ADMIN_URL}}/meta/types/user` |
| Users in Group | `GET` | `{{_flow.OKTA_URL}}/groups/{{_flow.SAMPLE_GROUP_ID}}/users` |

`Get Okta apps` is the only action with real pagination wired up (it follows the `rel="next"` link in the response's `Link` header). The rest return a single page and would need pagination added if a customer's org returns more results than one page.

## Required Customer Configuration

Hard-coded values that must be reviewed before running these actions:

| Value in template | Where | Customer action |
| --- | --- | --- |
| `https://{yourOktaDomain}-admin.okta.com/api/v1` | `OKTA_ADMIN_URL` flow variable | Replace with the customer's Okta admin API base URL. |
| `https://{yourOktaDomain}.okta.com/api/v1` | `OKTA_URL` flow variable | Replace with the customer's Okta org API base URL. |
| _(empty)_ | `SAMPLE_APP_ID` flow variable | Set to an Okta app ID if using `Get Users in App` / `Get Groups for app`, otherwise leave empty and disable/remove those actions. |
| _(empty)_ | `SAMPLE_GROUP_ID` flow variable | Set to an Okta group ID if using `Users in Group`, otherwise leave empty and disable/remove that action. |

## Credentials

- `custom-header-okta-read-token` (Custom Header): sets the `Authorization` header sent with every action above. Use an Okta API token in `SSWS <token>` format (**Security > API > Tokens** in the Okta admin console), scoped to read-only access where possible.

## Notes

- This template is a starting point, not a finished import pipeline: it has no schedule, no Jira Assets object-type mapping, and several actions are single-page reads that assume small result sets. Build out pagination, scheduling, and the Jira Assets `POST`/import-source actions per customer before enabling.
- Any Okta API token previously embedded directly in this template's actions (rather than referenced via the `custom-header-okta-read-token` credential) should be treated as compromised and rotated in Okta, since this repository is public.

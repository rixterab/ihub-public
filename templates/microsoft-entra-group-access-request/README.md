# Microsoft Entra Group Access Request

This template implements a one-way Jira-to-Microsoft-Entra access-request process. The `Microsoft Entra Group Access Request` flow reacts to created and updated Jira work items in the configured project, requires the request to be in the configured approved status, and adds or removes every user selected in a Jira multi-user field from one approved Entra group.

The template does not synchronize later Entra membership changes back to Jira.

## What The Template Does

1. Receives a Jira work item created or updated event and verifies project `AR`, status `Approved`, and an empty processing marker.
2. Iterates `customfield_10170`, the Jira multi-user picker, and resolves each Atlassian `accountId` with `{{lookupEmail accountId}}`.
3. Uses that email to look up the corresponding Entra user and captures the Entra object ID.
4. Reads the Entra group object ID from the selected option value in `customfield_10171`.
5. Reads `Add` or `Remove` from the action dropdown in `customfield_10172` and calls the matching Microsoft Graph membership endpoint.
6. Writes `Processed by iHub` to `customfield_10173` after a successful membership call so webhook redelivery or a later Jira update cannot repeat the request.

The created-event and updated-event action chains are intentionally separate because repository-proven conditions are AND-only and there is no proven OR condition operator.

## API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Find Entra user | `GET` | `https://graph.microsoft.com/v1.0/users/{{lookupEmail accountId}}?$select=id,userPrincipalName,mail` |
| Add group member | `POST` | `https://graph.microsoft.com/v1.0/groups/{group-id}/members/$ref` |
| Remove group member | `DELETE` | `https://graph.microsoft.com/v1.0/groups/{group-id}/members/{user-id}/$ref` |
| Mark request processed | `PUT` | `/rest/api/3/issue/{issue-key}` |

The Graph add body uses the required `@odata.id` reference to the user's directory object. Graph returns an error if an Add request targets an existing member or a Remove request targets a user who is not a direct member. Membership writes therefore have retries disabled; automatic retries could turn a successful but ambiguously delivered request into a misleading duplicate error.

## Body Format And Fidelity

No Jira rich text crosses the integration boundary. Jira user selections are reduced to account IDs and then email addresses. The group dropdown option value must be the immutable Entra group object ID, not the display name. The action dropdown values must be exactly `Add` and `Remove`.

`lookupEmail` is resolved server-side before the request is rendered. It accepts a plain field reference to an Atlassian account ID; within the user-picker iterator, the current item exposes that reference as `accountId`. A missing account ID or failed Jira lookup produces a blank string, after which the Graph lookup fails and no membership child action runs.

## Credential

Create the `oauth2-microsoft-entra-group-access` credential from an Entra app registration:

1. Register an application in the customer Entra tenant.
2. Add Microsoft Graph **application** permissions `GroupMember.ReadWrite.All` and `User.Read.All`.
3. Grant tenant-wide admin consent.
4. Create a client secret and record its value immediately.
5. In the iHub credential, replace `{tenant}` in the authorization and token URLs with the Entra tenant ID. Enter the application's client ID and the client secret value. The credential uses `client_credentials` and the scope `https://graph.microsoft.com/.default`; do not paste `Bearer` into either value.

The Jira write-back actions use the iHub Atlassian token. The Jira connection used by iHub must be able to browse project `AR`, resolve selected users' email addresses, and edit the processing-marker field.

## Jira Setup

1. Create or reuse project `AR` and an access-request issue type.
2. Create a **multi-user picker** named, for example, `Users`, and place it on the request and view screens.
3. Create a **single-select** field named, for example, `Entra Group`. Create one option per allowed group, with the option value set to the group's Entra object ID. Restricting the options is the authorization boundary; do not let requesters enter arbitrary group IDs.
4. Create a **single-select** field named `Action` with exactly two values: `Add` and `Remove`.
5. Create a text field named, for example, `Entra Request Processed`. Keep it off the requester-facing form and make it editable by the iHub integration user.
6. Replace the sample custom-field IDs in the flow with the IDs from this Jira site.
7. Configure the Jira event source for both work item created and work item updated events. Use the sample JQL `project = AR AND status = Approved`. The flow repeats the project and status checks so it fails closed if the event-source JQL is broadened later.
8. Ensure the approval workflow only enters `Approved` after the users, group, and action have been validated. Clear the processing field only when an operator intentionally wants to replay the request.

## Required Customer Configuration

| Placeholder | Sample | Required value |
| --- | --- | --- |
| `SPACE` | `AR` | Jira project key containing access requests. |
| `APPROVED_STATUS` | `Approved` | Exact Jira status name that authorizes execution. |
| `customfield_10170` | Users | Multi-user picker containing Atlassian users to change. |
| `customfield_10171` | Entra Group | Single-select field whose selected option value is an Entra group object ID. |
| `customfield_10172` | Action | Single-select field with exact values `Add` and `Remove`. |
| `customfield_10173` | Entra Request Processed | Text field used as the idempotency marker. |
| `{tenant}` | none | Entra tenant ID in both OAuth2 URLs in `manifest.json`. |
| `oauth2-microsoft-entra-group-access` | none | App client ID and client secret with the documented Graph application permissions. |

### Places To Update The Jira Custom Fields

- `customfield_10170`: both root action iterators.
- `customfield_10171`: all four Graph membership URLs.
- `customfield_10172`: all four Add/Remove action conditions.
- `customfield_10173`: both root deduplication conditions and all four Jira write-back bodies.

## Security

Only administrators should control the Entra Group field options. A requester who can alter an option value could target a different group with the app registration's authority. Apply Jira field configuration, context, workflow validators, and project permissions so the approved values cannot be changed after approval without a new approval cycle.

The Entra app uses application permissions and acts without a signed-in user. Grant it only the documented Graph permissions, protect the client secret, rotate it according to policy, and review iHub and Entra audit logs. Consider an Entra custom role or administrative unit if the tenant design supports narrower group-management scope.

## Limitations

- The group field is intentionally single-select. Supporting several groups in one request needs a verified nested-iterator or fan-out pattern; this repository does not currently prove that engine behavior.
- Jira-to-Entra email matching assumes the Jira email equals either the Entra user principal name or another identifier accepted by `GET /users/{id-or-userPrincipalName}`. If the directories use different addresses, add a maintained identity mapping instead of guessing.
- Jira email visibility and API access depend on the Atlassian integration user's permissions and the site's privacy configuration. If the email endpoint does not return an address, the Entra lookup cannot run.
- With several selected users, the processing marker is written after each successful membership call. If one user fails after another succeeds, the issue may already be marked processed. Inspect the action log, correct the failing identity, clear the marker deliberately, and account for already-applied memberships before replaying.
- The flow manages direct membership only. It does not handle dynamic groups, transitive membership, role-assignable groups, group ownership, access reviews, entitlement-management access packages, or time-limited membership.
- It does not transition or comment on the Jira request. Add those as explicit child actions only after deciding how partial failures should be represented.

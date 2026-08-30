# Trello - Create Card from Jira

This template installs one iHub flow that creates a Trello card whenever a work item is created in a configured Jira project.

- `Trello - Create Card from Jira`: Jira -> Trello, triggered by Jira webhooks.

The integration is one-way. Nothing comes back from Trello into Jira. See [Limitations](#limitations).

## What The Template Does

- Listens for Jira `avi:jira:created:issue` events.
- Ignores every work item that is not in the Jira project configured in `SPACE`.
- Ignores work items that already carry a Trello card ID, so a redelivered webhook never creates a second card.
- Creates one Trello card at the top of the configured list.
- Writes the card ID back to a Jira custom field, which is what makes the previous step work.
- Attaches the Jira work item URL to the card. Attachments are Trello's mechanism for linking external resources, the same approach `linear-sync` takes with Linear.

## Trello API Calls

| Action | Method | Endpoint |
| --- | --- | --- |
| Create Trello Card | `POST` | `https://api.trello.com/1/cards` |
| Store Trello Card ID on Jira Issue | `PUT` | `{{baseUrl}}/rest/api/3/issue/{{issue.key}}` |
| Attach Jira Link to Trello Card | `POST` | `https://api.trello.com/1/cards/{id}/attachments` |

The create action stores `$.response.data.id` in `trelloCardId` and `$.response.data.shortUrl` in `trelloCardUrl`. Store the **`id`** (a 24-character hex string), not `idShort`: every Trello card endpoint takes the full ID, and that is the value written back to Jira.

`TRELLO_LIST_ID` is a list ID, not a board ID. Read it by appending `.json` to the board URL and finding the list, or from `GET /1/boards/{boardId}/lists`.

## Body Format And Fidelity

Trello card descriptions are Markdown. The template passes the Jira ADF description through `adfToHTML`, matching `github-issues-sync`. Trello's Markdown renderer strips most inline HTML, so formatting is reduced further than it is in GitHub or GitLab — the text survives, the styling largely does not. The link back to Jira at the top of the description is the reliable affordance.

Free-text Jira values are interpolated with triple braces so an `&` or an apostrophe is not turned into an HTML entity. As in every other template here, a double quote inside a Jira summary can still break the surrounding JSON body.

## Credential

The manifest defines one credential: `custom-header-trello-auth`, of type `CUSTOM_HEADER` with a single `Authorization` header.

Trello documents key and token as query parameters, which would put secrets in the flow URL and the logs. It also accepts them in an `Authorization` header, which is what this template uses. Get a key and token from `https://trello.com/power-ups/admin` (create a Power-Up, then generate an API key and authorise a token), and paste the header value in this exact form:

```
Authorization: OAuth oauth_consumer_key="<your key>", oauth_token="<your token>"
```

The quotes and the comma-space are part of the format; Trello rejects the header without them.

## Required Customer Configuration

Create a Jira custom field of type single-line text to store the Trello card ID. The template uses `customfield_10153`; every customer must replace this with the field ID from their Jira instance.

| Value in template | Where | Customer action |
| --- | --- | --- |
| `TRELLO` | `SPACE` flow variable | Replace with the Jira project key that should create Trello cards. |
| `000000000000000000000000` | `TRELLO_LIST_ID` flow variable | Replace with the target Trello list ID. |
| `customfield_10153` | Flow JSON, two places | Replace with the customer's Jira custom field key. |
| Empty `Authorization` header | `custom-header-trello-auth` credential | Set to the `OAuth oauth_consumer_key="…", oauth_token="…"` value. |

### Places To Update The Jira Custom Field

- `Create Trello Card` condition: checks `$.issue.fields.customfield_10153` is empty before creating.
- `Store Trello Card ID on Jira Issue` body: writes the created ID into `customfield_10153`.

## Limitations

- One-way. Card moves, comments, checklists and archiving are not written back to Jira. The other direction needs a second flow on a `CUSTOM_WEBHOOK_TRIGGER_TYPE` fed by a Trello board webhook; the ID write-back here is the link it would search on.
- Create only. Later Jira updates, comments and attachments are not synced to the card.
- No label or member mapping. Trello labels are per-board and Trello members have no shared key with Jira accounts.
- File contents are not copied. The Jira attachment stays in Jira.
- One Jira project and one Trello list per flow.

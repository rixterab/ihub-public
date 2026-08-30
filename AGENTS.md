# AGENTS.md

Working guide for building iHub integration templates in this repository. Written for
agents, useful for humans. Read this before creating or editing anything under
`templates/`.

The single most important rule: **this repository is its own best documentation.**
Before writing a new template, open the closest existing one and copy its shape. Every
convention below was derived from templates that already work. When this guide and the
repository disagree, the repository wins — and the guide should be updated.

---

## 1. What a template is

One folder under `templates/`, containing:

```
templates/<kebab-case-key>/
├── manifest.json          describes the template
├── <flow_name>.json       one or more exported iHub flows (snake_case)
├── README.md              how to install, configure and reason about it
└── <icon>.svg             the product mark
```

The folder name, the manifest `key`, and the README title should agree.

### Naming

The name states the direction, because direction is the thing a reader most needs to
know:

| Shape | Folder | Example |
| --- | --- | --- |
| Jira → tool, one-way | `<tool>-<noun>-from-jira` | `gitlab-issue-from-jira` |
| Tool → Jira, one-way | `jira-<noun>-from-<tool>` | `jira-incident-from-datadog` |
| Two-way | `<tool>-sync` | `linear-sync`, `servicenow-sync` |
| Scheduled pull into Jira Assets | `<tool>-asset-import` | `jamf-asset-import` |
| Signature flows | `jira-<vendor>-signature` | `jira-docusign-signature` |

---

## 2. Pick a reference template first

Do not start from scratch. Start from whichever of these is closest, and keep its
action graph:

| You are building | Copy | Why |
| --- | --- | --- |
| Jira → tool, create something | `gitlab-issue-from-jira` | Create + guard + write-back |
| Full two-way sync | `github-issues-sync`, `linear-sync` | Loop guards, comments, attachments |
| Tool → Jira from a webhook | `splunk-jira-incident-alerts` (incoming flow) | Scripted variables, JQL dedup, create-or-comment |
| Scheduled pull into Jira Assets | `microsoft-entra-asset-import` | The seven-action import-source dance |
| Chat notification | `slack-notify-work-item-created` | mrkdwn rules, `ok:false` handling |
| Scheduled poll of a REST API | `snyk-findings-to-jira` | Pagination + iterator + per-item dedup |

`linear-sync/README.md` is the best-documented template in the repository. Use it as
the bar for README quality, not as a length target.

---

## 3. Flow file anatomy

```jsonc
{
  "actions": [ /* see below */ ],
  "description": "One line, matches the manifest description",
  "enabled": false,              // ALWAYS false. Never ship an enabled flow.
  "flowVariables": [ /* customer configuration */ ],
  "groupName": "Vendor",
  "logLevel": "OFF",
  "logRetention": 3,
  "metadata": { "createdBy": "", "updatedBy": "" },   // must be blank; see below
  "name": "Human Readable Flow Name",
  "trigger": { "type": "..." },
  "useStaticIP": false
}
```

### Triggers

| Type | Use |
| --- | --- |
| `ATLASSIAN_WEBHOOK_TRIGGER_TYPE` | Jira events. Payload has `issue`, `eventType`, `baseUrl`. |
| `CUSTOM_WEBHOOK_TRIGGER_TYPE` | Anything calling iHub. Add a `config.response` block when the caller needs a fast fixed answer (Slack needs `200` within 3s). |
| `SCHEDULED_TRIGGER_TYPE` | Polls and asset imports. `"value": "0 9 * * *"`. |
| `JWT_WEBHOOK_TRIGGER_TYPE` | One use, in `google-add-work-item-to-sheets-jwt`. Do not reach for it without reading that template. |

### Actions

Every action is `WEB_REQUEST_ACTION_TYPE` — there is no other action type in this
repository. Required keys:

```jsonc
{
  "headers": [ /* the six standard headers, see below */ ],
  "method": "POST",
  "name": "Create GitLab Issue",
  "description": "Create a GitLab issue from a Jira work item.",  // required, one sentence
  "credentialsReference": { "type": "DATABASE_REF", "value": "template:<credential-key>" },
  "id": "template:action:<uuid4>",
  "type": "WEB_REQUEST_ACTION_TYPE",
  "body": "…",                     // a JSON *string*, escaped
  "logConditionsNotMet": false,
  "enabled": true,
  "url": "…",
  "parentId": "",                  // "" for root, else another action's full id
  "flowVariables": [ /* values captured from the response */ ],
  "conditions": [ /* one COMBINED_CONDITION wrapping the list */ ],
  "order": 100                     // 100, 200, 300 … unique within the flow
}
```

- `order` is display order and must be unique. `parentId` is what actually sequences
  execution. They do not have to agree — in the asset-import templates `Submit Data`
  has `order: 200` but is a child of the action at `order: 600`.
- Generate a **fresh UUID4** for every action id. Never copy an id from another
  template. (54 duplicate ids already exist across old copied templates; do not add
  more.)
- Credentials: `{"type": "ATLASSIAN_TOKEN", "value": "ATLASSIAN_AUTH_TOKEN"}` for Jira
  calls, `{"type": "DATABASE_REF", "value": "template:<key>"}` for everything else,
  where `<key>` must exist in the manifest's `credentialDefinitions`.

### The six standard headers

Every action, every template, in this order:

```json
[
  { "value": "application/json", "key": "Accept" },
  { "value": "application/json", "key": "Content-Type" },
  { "value": "keep-alive", "key": "Connection" },
  { "value": "gzip, deflate, br", "key": "Accept-Encoding" },
  { "value": "IntegrationsHub/Cloud", "key": "User-Agent" },
  { "value": "no-cache", "key": "Cache-Control" }
]
```

Vendor-specific headers (`X-GitHub-Api-Version`, `Prefer`, `From`) go after
`Content-Type`. Change `Accept` or `Content-Type` when the API demands it — Slack
needs `application/json; charset=utf-8`, PagerDuty needs
`application/vnd.pagerduty+json;version=2`.

### Conditions

Wrap the list in a single `COMBINED_CONDITION`. Entries are ANDed.

```json
{
  "jsonPath": "$.eventType",
  "type": "DATA_CONDITION",
  "value": "avi:jira:created:issue",
  "operator": "EQUALS",
  "description": "Only run for Jira work item created events"
}
```

**Only four operators are proven in this repository**: `EQUALS`, `NOT_EQUALS`,
`EMPTY_ARRAY`, `NOT_EMPTY_ARRAY`. If you need set membership or a comparison, push the
filter into the API request instead of inventing an operator — that is what
`snyk-findings-to-jira` does with its severity query fragment.

`value` may contain `{{_flow.X}}`; that is how configurable conditions are built. Always
write a `description` — a condition without one is unreadable in the iHub UI.

### Flow variables

```json
{ "name": "GITLAB_PROJECT_ID", "behavior": "SET", "value": "12345678",
  "instruction": "GitLab project ID or URL-encoded namespace/project path." }
```

- Flow-level `flowVariables` = customer configuration. `UPPER_SNAKE_CASE`.
- Action-level `flowVariables` = values captured from that action's response, with a
  JSONPath in `value`. `camelCase`.
- `behavior` is always `SET`.
- `instruction` is not optional. It is what the customer reads in the UI.
- `SPACE` is the conventional name for "the Jira project key this flow watches".

**Anything a customer must change belongs in a flow variable**, not hardcoded in a
condition or a URL. A README row saying "edit the action condition" is a smell.

---

## 4. Templating rules

iHub renders Handlebars. The escaping rule is the single most common source of bugs.

| Form | Behaviour | Use for |
| --- | --- | --- |
| `{{x}}` | HTML-escapes `& < > " '` | Slack mrkdwn, Teams Adaptive Cards, controlled values (keys, enum names) |
| `{{{x}}}` | Raw | Free text going into a JSON body: summary, description, priority name, reporter name |

Getting this wrong is silent: `{{issue.fields.summary}}` in a JSON body puts
`&#x27;` and `&amp;` into the created record. Slack and Teams are the deliberate
exceptions — their READMEs explain why.

Known trade-off, accepted repository-wide: a double quote inside a Jira summary can
still break a JSON body under `{{{ }}}`. Say so in the README rather than pretending
otherwise.

### Helpers

Three exist. Do not invent others.

| Helper | Does | Note |
| --- | --- | --- |
| `{{{adfToHTML x}}}` | Jira ADF → HTML | For descriptions and comments going *out* |
| `{{{htmlToADF x}}}` | HTML → Jira ADF | For descriptions coming *in*; emits a JSON object, so use it **unquoted** |
| `{{{toJSON x}}}` | Value → JSON literal | Also unquoted |

`issue.fields.description` on Jira Cloud is an **ADF object, not a string**. Never
interpolate it bare — you get `[object Object]` or a broken body. Either run it through
`adfToHTML`, or omit it and link back to Jira (which is what the Slack template does
deliberately).

Target format matters: `adfToHTML` output is HTML. GitHub and GitLab render inline HTML
inside Markdown, so it works there. Bitbucket needs `"markup": "html"` alongside it.
Trello mostly strips it. Linear rejects it outright, which is why `linear-sync` uses
scripted variables to produce plain text instead. Pick per target and say what you
picked.

### Scripted variables

For incoming webhooks whose payload shape you do not fully control. Read the payload
tolerantly, and always with the same prelude the repository already uses:

```js
const root = scope && scope.payload && typeof scope.payload === 'object' ? scope.payload : scope;
const at = (o, p) => { /* dotted-path getter, array-index aware */ };
const read = (...ps) => { /* first non-empty path wins */ };
```

Then one variable per derived value, each returning a **string**. Reference them in
actions and conditions without the `_flow.` prefix: `{{alertKey}}`, `$.alertKey`.

Normalise vendor states to a fixed vocabulary (`firing` / `resolved` / `acknowledged`)
in a scripted variable, then branch on that — never branch on the raw vendor string in
five different places.

### Pagination

```json
"pagination": {
  "variable": { "name": "PAGE_URL", "script": "…returns next URL or the seed URL…" },
  "breakJsonPath": "$['response']['data']['@odata.nextLink']",
  "type": "RESPONSE_BASED",
  "enabled": true
}
```

The URL then becomes `{{{PAGE_URL}}}`. `RESPONSE_BASED` works when the API returns a
next-page URL or cursor. `CUSTOM` gives the script the whole `response` object, for
header-based (`Link: …; rel="next"`) or token-based paging.

**APIs that page by number are the hard case.** The script is stateless and cannot see
its own previous value, so there is no way to increment a counter. Either find a
cursor, or fetch a single large page and document the ceiling — do not ship a
pagination block that silently loops on page 0.

---

## 5. The patterns that matter

### Idempotency and write-back

**Webhooks get redelivered. A create action with no guard creates duplicates.** The
pattern, in every Jira → tool template that creates something:

1. Create action: condition `$.issue.fields.customfield_XXXXX` `EQUALS` `""`.
2. Child action: `PUT {{baseUrl}}/rest/api/3/issue/{{issue.key}}` writing the external
   ID into that field, conditioned on the captured variable being non-empty.

For tool → Jira flows, invert it: search first, then create or comment.

```
POST {{baseUrl}}/rest/api/3/search/jql
{ "jql": "project = {{_flow.SPACE}} AND \"cf[10155]\" ~ \"{{alertKey}}\" AND statusCategory != Done" }
```

then one child on `EMPTY_ARRAY` (create) and one on `NOT_EMPTY_ARRAY` (comment).

Conventional custom fields: `customfield_10153` for a linked-object ID,
`customfield_10155` for an external alert key, `customfield_10156` for a security
finding ID. These are **placeholders** — every README must tell the customer to replace
them, and list every place they appear.

### Loop prevention

Any two-way sync needs all of these, and the README must enumerate them:

1. Actor filter on the Jira side (`$.comment.author.accountId` `NOT_EQUALS` the
   integration account).
2. Actor filter on the tool side.
3. Create only when the link is absent.
4. Jira issue properties (`ihub-<external-id>`) to suppress already-synced comments.
5. Awareness that the write-back is itself a Jira update that fires a webhook.

Also check for loops *across* templates. Installing `pagerduty-incident-from-jira` and
`jira-incident-from-pagerduty` against the same service is a cycle; that README says so.

### Errors that are not HTTP errors

Several APIs answer `200` on failure. Capture the real result into flow variables and
say so in the README:

- Slack: `{"ok": false, "error": "channel_not_found"}`
- GraphQL (Linear): `200` with an `errors` array; Linear signals rate limits as `400`
- New Relic Event API: accepts then validates asynchronously — failures never reach the
  flow log at all

### Rate limits

Add a `ratelimit` block to outbound calls against third-party APIs:

```json
{ "sleepTime": 500, "initialSleepTime": 5000, "maxRetries": 0, "burst": 20,
  "retryStatusCodes": [408, 429, 500, 502, 503, 504] }
```

That is the most common shape in the repository. Two deliberate variations exist and
both are worth understanding before copying either:

- `maxRetries: 3` where the API is safe to retry.
- `maxRetries: 0` **with `400` added to `retryStatusCodes`** in `linear-sync`, because
  Linear signals rate limiting as `400` with a `RATELIMITED` code in the body rather
  than as `429`. Retrying there would also retry genuinely malformed requests, which is
  why retries are off.

Lower `burst` for paging APIs. Jira write-back actions do not carry a `ratelimit` block
at all.

### Jira Assets imports

Seven actions, in this shape (copy `microsoft-entra-asset-import`):

| order | action | endpoint |
| --- | --- | --- |
| 100 | Update Import Mapping | `PATCH …/importsource/{id}/mapping` |
| 300 | Check Import Status | `GET …/configstatus` |
| 400 | Cancel Stuck Import | `DELETE {{cancelUrl}}` when status is `RUNNING` |
| 500 | Start Import | `POST …/executions` |
| 600 | Fetch data | vendor API, paginated |
| 200 | Submit Data | `POST {{submitResultsUrl}}` |
| 700 | Complete Import | `POST {{submitResultsUrl}}` `{"completed": true}` |

`{{context.workspaceId}}`, `{{submitResultsUrl}}` and `{{cancelUrl}}` are supplied by
iHub. The mapping body needs both `schema.objectSchema.objectTypes[].attributes[]` and
`mapping.objectTypeMappings[]`; exactly one attribute carries `"externalIdPart": true`
and exactly one carries `"label": true`. Referenced object types get
`"objectMappingIQL": "Name Like ${locator}"`.

---

## 6. manifest.json

```json
{
  "schemaVersion": 1,
  "key": "gitlab-issue-from-jira",
  "name": "GitLab - Create Issue from Jira",
  "description": "Create a GitLab issue when a Jira work item is created",
  "category": "GitLab",
  "flows": ["gitlab_issue_from_jira.json"],
  "icon": "gitlab.svg",
  "author": "Rixter AB",
  "tags": ["gitlab", "devops", "issues", "work-items"],
  "createdAt": "2026-08-30",
  "credentialDefinitions": [ … ]
}
```

`flows` and `icon` must name files that exist. Every `credentialDefinitions[].key` must
be referenced by at least one action, and every `template:<key>` reference must resolve.

`credentialDefinitions` is **optional**: six templates omit it entirely (`workday`,
`bamboohr`, `personio`, `hibob`, `factorial`, `deel`), because every action they run is
a Jira call using `ATLASSIAN_TOKEN`. Omit the key rather than shipping an empty array
when nothing external is called.

### Credential types

| Type | Sends | Use when |
| --- | --- | --- |
| `TOKEN` | `Authorization: Bearer <token>` | Plain bearer tokens (GitHub, HubSpot, Airtable) |
| `CUSTOM_HEADER` | A named header you define | Anything else: `PRIVATE-TOKEN`, `DD-API-KEY`, `Circle-Token`, or `Authorization` with a non-Bearer prefix |
| `BASIC_AUTH` | Basic auth | Username + app password / API token |
| `OAUTH2` | Managed OAuth2 | Refreshable access; supports `code` and `client_credentials` |
| `JWT`, `AWS_SIGNATURE` | Specialised | See `google-*`, `aws-*` |

Prefer `TOKEN` over `CUSTOM_HEADER` + "paste `Bearer …` yourself". Prefer `OAUTH2` over
a pasted access token that expires within the hour.

The README must state the **exact value to paste**, including or excluding the prefix.
This is where customers get stuck: PagerDuty wants `Token token=…`, Linear wants the
raw key with no prefix, Slack wants `Bearer xoxb-…`, Trello wants
`OAuth oauth_consumer_key="…", oauth_token="…"`.

---

## 7. README.md

Match `linear-sync/README.md` and `slack-notify-work-item-created/README.md`. Sections,
in order:

1. **Title and one-paragraph summary**, naming each flow and its direction.
2. **State one-way vs two-way explicitly**, in the first few lines.
3. **What The Template Does** — bullets, in execution order.
4. **API Calls** — a table of action → method → endpoint, plus anything surprising
   about the API (required headers, `200`-on-failure, ID vs IID, deprecations).
5. **Body Format And Fidelity** — when text crosses a format boundary, say what is lost
   and why you chose that trade-off.
6. **Credential** — where to create it, what scopes, and the exact string to paste.
7. **Vendor-side setup** — numbered steps, including the webhook payload template when
   the vendor's payload is user-defined (as Datadog's is).
8. **Required Customer Configuration** — a table of *every* placeholder: flow
   variables, custom fields, credential values. Plus a "Places To Update The Jira Custom
   Field" list when a field appears in several actions.
9. **Security** — for any `CUSTOM_WEBHOOK_TRIGGER_TYPE` flow. See below.
10. **Limitations** — what it does not do, and what building the missing part would
    take.

The Limitations section is not a disclaimer, it is the most useful part of the document.
Write it honestly and specifically: "one-way only; the incoming direction needs a second
flow on a custom webhook trigger fed by a GitLab project webhook, and the IID write-back
in this template is the link it would search on."

### Webhook security boilerplate

**iHub cannot verify webhook signatures.** Every template with a
`CUSTOM_WEBHOOK_TRIGGER_TYPE` must say so plainly, and must tell the customer to treat
the iHub webhook URL as a credential: do not commit it, paste it into tickets or chat,
or share it; recreate the trigger to rotate it if it leaks. Where the flow can do a weak
authenticity check (filtering on an expected org or team ID), do it and say it is not a
substitute.

If a flow can transition work items from a chat click, also say that chat visibility is
not a Jira permission — the flow acts as the integration user and bypasses Jira's own
approval permissions.

---

## 8. Before you commit

Run this against the templates you touched:

```python
# JSON parses; manifest flows/icon exist; credential refs resolve; parentId targets
# exist; every {{_flow.X}} is defined; bodies parse as JSON with placeholders stripped;
# orders unique; action ids unique; every flow variable, custom field and credential
# key appears in the README.
```

Checker notes learned the hard way:

- Substituting every `{{…}}` with `X` reports false failures on `{{{htmlToADF x}}}` and
  `{{{toJSON x}}}`, which are *unquoted* JSON insertions. Replace triple-brace forms
  with `null` and quoted forms with `"X"`.
- Repo-wide duplicate action ids exist in old templates. Check that *your* ids are new
  rather than that the repo has none.

Manual checks a script cannot do:

- Does the flow ship `"enabled": false` and blank `metadata`?
- Would a redelivered webhook create a duplicate?
- Does every free-text interpolation into a JSON body use `{{{ }}}`?
- Is every value a customer must change a flow variable, and is it in the README table?
- Have you actually verified the payload paths against a real delivery, or are they
  assumptions? If assumptions, say so in the README.

---

## 9. Do not

- **Do not ship secrets.** No tokens, API keys, webhook URLs, customer hostnames,
  account IDs, email addresses or real data. Placeholders only.
- **Do not ship an enabled flow.** `"enabled": false` at the flow level, always.
- **Do not ship populated `metadata.createdBy` / `updatedBy`.** Those fields hold
  Atlassian account IDs. Twelve older templates still carry one (`zoho-*`, `aws-*`,
  `microsoft-entra-asset-import`, `create-google-group`, `remove-user-in-google`) —
  that is a defect to fix, not a precedent to copy.
- **Do not copy action UUIDs** from another template.
- **Do not invent condition operators, helpers or field names.** If the construct has no
  precedent in `templates/`, either find another way or flag it prominently as
  unverified.
- **Do not interpolate `issue.fields.description` bare.** It is ADF.
- **Do not use `{{ }}` for free text in a JSON body**, or `{{{ }}}` in Slack mrkdwn.
- **Do not create without a dedup guard.**
- **Do not hardcode a customer-specific value in a condition** and then tell the reader
  to go edit the condition. Make it a flow variable.
- **Do not claim a direction the template does not implement.** A one-way template says
  so, in the summary and in Limitations.
- **Do not write a thin README.** A config table alone is not a README; see section 7.
- **Do not invent logo SVG path data.** Copy an existing icon from another template when
  the vendor already appears in the repository, or ship a plain coloured mark with the
  product initial and note in the README that it is a placeholder to be swapped for the
  official asset.
- **Do not guess at engine semantics.** If you do not know whether iHub supports
  something, look for an existing template that does it. If none does, choose the
  approach you can verify and document the limitation.

---

## 10. Quick start

1. Decide the direction and pick the reference template from section 2.
2. `cp -r templates/<reference> templates/<new-key>`.
3. Regenerate every action `id` as a fresh UUID4.
4. Rewrite `manifest.json`: key, name, description, category, flows, icon, tags,
   `createdAt`, credential definitions.
5. Rewrite the flow: trigger, flow variables, URLs, bodies, conditions. Keep the six
   standard headers, `logConditionsNotMet`, `"enabled": false` and blank `metadata`.
6. Add the dedup guard and the write-back, or the search-then-create-or-comment pair.
7. Add the icon.
8. Write the README against the section 7 outline.
9. Run the section 8 checks.
10. Open a PR describing the template and its intended use case.

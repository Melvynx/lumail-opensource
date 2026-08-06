---
name: lumail
version: "0.4.0"
tags: ["email", "marketing", "newsletter", "mcp", "api", "cli"]
description: Manage Lumail email marketing via CLI. Use for subscribers, campaigns, tags, workflows, segments, transactional emails, analytics, TipTap campaign content, or Lumail API tools.
---

# Lumail - Email Marketing CLI

Interact with the Lumail API using `npx lumail`. The CLI exposes the most-used resources directly, plus a generic `tools` runner that gives access to up to 96 organization-aware tools used by AI agents.

## Quick Start

```bash
npx lumail auth login             # Sign in through the browser (recommended)
npx lumail auth test              # Verify it works
npx lumail tools list             # See every tool enabled for this organization
```

`auth login` runs the OAuth flow: it registers a public client, opens the
browser with a PKCE challenge, waits on a loopback callback, and stores the
credential in `~/.config/lumail/oauth.json` (mode 0600). Expired tokens refresh
automatically before each command. **Prefer it — do not ask the user for an API
token unless they bring one up themselves.**

An API token still works for CI and for named accounts:

```bash
npx lumail auth set <api-key>     # Save token (~/.config/lumail/token)
```

For the MCP connection installed by the Lumail plugin, see the `lumail-plugin`
skill instead — that one needs no credential at all.

## Named Accounts

Lumail supports named API-key accounts so one machine can target multiple organizations without overwriting the default token.

**Use named accounts whenever the user names an account or org slug, for example `thumbfast`. Do not swap `~/.config/lumail/token` manually.**

When the current URL contains `/orgs/<orgSlug>/`, use `--account <orgSlug>` for every CLI call. Run `lumail accounts list` before the first mutation. Never silently fall back to another named account: API tokens are organization-scoped, so the wrong account can hide feature-gated tools or target the wrong data.

If the requested account is missing but the default token is already authenticated for that organization, save it explicitly before continuing:

```bash
TOKEN=$(tr -d '\n' < ~/.config/lumail/token)
lumail accounts add <orgSlug> "$TOKEN"
```

If `configure_workflow_v2_draft` returns `Tool not found`, first compare the default catalog with the named account catalog. The tool is intentionally hidden when Workflow V2 is disabled for the token's organization.

Named accounts are stored in `~/.config/lumail/accounts.json`:

```json
{
  "accounts": {
    "thumbfast": "lum_..."
  }
}
```

### Account-aware CLI

Use `-a` / `--account` on the command or subcommand:

```bash
lumail accounts add thumbfast <api-key>
lumail tools run get_campaign --account thumbfast --params '{"campaignId":"cmp_..."}' --json
lumail tools run edit_campaign --account thumbfast --params '{"id":"cmp_...","operations":[{"op":"replace_text","search":"old","replace":"new","all":true}]}' --json
```

For Thumbfast-specific Lumail work, use:

```bash
lumail tools run get_campaign --account thumbfast --params '{"campaignId":"<campaignId>"}' --json
lumail tools run edit_campaign --account thumbfast --params '{"id":"<campaignId>","subject":"New subject"}' --json
```

### Stale Installed CLI Fallback

Some installed `npx lumail`, global `lumail`, or `lumail-cli` binaries may be stale and may not expose `accounts` or `--account`. If `lumail --help` does not show `-a, --account <name>`, run the account-aware source CLI from the Lumail checkout with Bun instead:

```bash
cd /Users/melvynx/Developer/projects/lumail.io
bun src/cli/index.ts --help
bun src/cli/index.ts accounts add thumbfast <api-key>
bun src/cli/index.ts tools run get_campaign --account thumbfast --params '{"campaignId":"cmp_..."}' --json
bun src/cli/index.ts tools run edit_campaign --account thumbfast --params '{"id":"cmp_...","operations":[{"op":"replace_text","search":"old","replace":"new","all":true}]}' --json
```

This source CLI reads the same `~/.config/lumail/accounts.json` account store.

## Global Flags

All commands accept these:

| Flag                         | Description                                           |
| ---------------------------- | ----------------------------------------------------- |
| `--json`                     | Output as JSON                                        |
| `--format <text\|json\|csv>` | Output format (default: text)                         |
| `--verbose`                  | Enable debug logging                                  |
| `--no-color`                 | Disable colored output                                |
| `--no-header`                | Omit table headers (for piping)                       |
| `-a, --account <name>`       | Use a named account, when supported by the active CLI |

## Authentication

| Command                       | Description                              |
| ----------------------------- | ---------------------------------------- |
| `npx lumail auth set <token>` | Save API key to `~/.config/lumail/token` |
| `npx lumail auth show`        | Show masked token                        |
| `npx lumail auth show --raw`  | Show full token                          |
| `npx lumail auth remove`      | Delete saved token                       |
| `npx lumail auth test`        | Verify the token is valid                |

If multiple Lumail organizations are used on the same machine, prefer named accounts:

| Command                              | Description                                              |
| ------------------------------------ | -------------------------------------------------------- |
| `lumail accounts add <name> <token>` | Save a named API key in `~/.config/lumail/accounts.json` |
| `lumail accounts list`               | List configured named accounts with masked tokens        |
| `lumail accounts show <name>`        | Show a masked named account token                        |
| `lumail accounts show <name> --raw`  | Show the full named account token                        |
| `lumail accounts remove <name>`      | Delete a named account                                   |

## Resources (first-class CLI commands)

### subscribers

| Command                                                                                | Description              |
| -------------------------------------------------------------------------------------- | ------------------------ |
| `npx lumail subscribers list`                                                          | List subscribers         |
| `npx lumail subscribers list --tag newsletter --status SUBSCRIBED --limit 50`          | Filter by tag and status |
| `npx lumail subscribers list --query "john" --limit 20`                                | Search by name or email  |
| `npx lumail subscribers list --cursor <subscriberId>`                                  | Cursor pagination        |
| `npx lumail subscribers create --email user@example.com --name "John" --tags vip beta` | Upsert subscriber        |
| `npx lumail subscribers get user@example.com`                                          | By email or ID           |
| `npx lumail subscribers update user@example.com --name "John Doe" --tags premium`      | Append tags              |
| `npx lumail subscribers update user@example.com --tags premium --replace-tags`         | Replace tags             |
| `npx lumail subscribers delete user@example.com`                                       | Hard delete              |
| `npx lumail subscribers unsubscribe user@example.com`                                  | Unsubscribe              |
| `npx lumail subscribers add-tags user@example.com --tags vip premium`                  | Add tags                 |
| `npx lumail subscribers remove-tags user@example.com --tags old-tag`                   | Remove tags              |
| `npx lumail subscribers events user@example.com --take 50 --order desc`                | Recent events            |

Statuses: `SUBSCRIBED`, `UNSUBSCRIBED`, `BOUNCED`, `COMPLAINED`, `INACTIVE`, `UNCONFIRMED`.

### campaigns

| Command                                                                                              | Description                              |
| ---------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `npx lumail campaigns list`                                                                          | List campaigns                           |
| `npx lumail campaigns list --status DRAFT --page 1 --limit 50`                                       | Filter by status                         |
| `npx lumail campaigns list --query "welcome" --json`                                                 | Search by name/subject                   |
| `npx lumail campaigns create --subject "Welcome!" --name "Welcome" --preview "Welcome aboard"`       | Create draft                             |
| `npx lumail campaigns get <campaignId>`                                                              | Get details                              |
| `npx lumail campaigns update <campaignId> --subject "..." --preview "..."`                           | **Metadata only** (subject/name/preview) |
| `npx lumail campaigns delete <campaignId>`                                                           | DRAFT only                               |
| `npx lumail campaigns send <campaignId>`                                                             | Send immediately                         |
| `npx lumail campaigns send <campaignId> --scheduled-at 2026-01-15T10:00:00Z --timezone Europe/Paris` | Schedule                                 |

Statuses: `DRAFT`, `SCHEDULED`, `SENDING`, `SENT`, `ARCHIVED`, `FAILED`.

> **Important:** `campaigns update` does NOT touch the email body. To edit content, use `tools run edit_campaign` (see "Editing campaign content" below).

### tags

| Command                                     | Description   |
| ------------------------------------------- | ------------- |
| `npx lumail tags list`                      | List all tags |
| `npx lumail tags create --name "premium"`   | Create tag    |
| `npx lumail tags get premium`               | By name or ID |
| `npx lumail tags update <id> --name "gold"` | Rename        |

To delete a tag, use `tools run delete_tag --params '{"tagId":"..."}'` (dangerous action, requires confirmation).

### emails (transactional)

| Command                                                                                                                       | Description                  |
| ----------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| `npx lumail emails send --to user@example.com --from noreply@you.com --subject "..." --content "..." --content-type MARKDOWN` | Send                         |
| `npx lumail emails send ... --content-type HTML`                                                                              | HTML body                    |
| `npx lumail emails send ... --content-type TIPTAP --content '<JSON>'`                                                         | TipTap body                  |
| `npx lumail emails send ... --reply-to support@you.com --transactional`                                                       | With reply-to + tracking off |
| `npx lumail emails verify user@example.com`                                                                                   | Verify deliverability        |

### events

| Command                                                                                                                | Description    |
| ---------------------------------------------------------------------------------------------------------------------- | -------------- |
| `npx lumail events create --type SUBSCRIBER_PAYMENT --subscriber user@example.com --data '{"amount":99,"plan":"pro"}'` | Track an event |

Event types: `SUBSCRIBED`, `UNSUBSCRIBED`, `TAG_ADDED`, `TAG_REMOVED`, `EMAIL_OPENED`, `EMAIL_CLICKED`, `EMAIL_SENT`, `EMAIL_RECEIVED`, `EMAIL_BOUNCED`, `EMAIL_COMPLAINED`, `WORKFLOW_STARTED`, `WORKFLOW_COMPLETED`, `WORKFLOW_CANCELED`, `FIELD_UPDATED`, `WEBHOOK_EXECUTED`, `SUBSCRIBER_PAYMENT`, `SUBSCRIBER_REFUND`.

## V2 Tools API

`lumail tools` is a generic runner over the organization-aware tool API used internally by AI agents and the MCP server. Up to 96 tools are currently available; feature-specific tools such as Workflow V2 only appear for enabled organizations. Use it for anything the dedicated CLI commands do not cover (workflows, segments, edit_campaign content, analytics, etc.).

```bash
npx lumail tools list                                    # List enabled tools
npx lumail tools list --raw                              # Full schemas
npx lumail tools get <tool-name>                         # Inspect one tool's schema
npx lumail tools run <tool-name> --params '<json>'       # Invoke
```

### Tool catalog (categorized)

#### Subscribers (10 tools)

`list_subscribers` · `get_subscriber` · `add_subscriber` · `unsubscribe`\* · `bulk_add_tags` · `bulk_remove_tags` · `count_subscribers_by_status` · `get_subscribers_by_tag` · `get_subscriber_emails` · `list_subscriber_events` · `create_event` · `get_subscriber_stats` · `get_subscriber_growth`

#### Tags (4 tools)

`list_tags` · `create_tag` · `get_or_create_tags` · `delete_tag`\*

#### Campaigns (19 tools)

`list_campaigns` · `get_campaign` · `create_campaign` · **`edit_campaign`** · **`auto_fit_campaign_images`** · `duplicate_campaign` · `update_campaign_filters` · `get_available_filters` · `send_test_email` · `render_campaign` · `schedule_campaign`\* · `cancel_campaign_schedule`\* · `get_campaign_progress` · `get_campaign_analytics` · `list_campaign_history` · `get_campaign_history_entry` · `restore_campaign_history` · `archive_campaign` · `unarchive_campaign`

#### Workflows

Workflow V1: `list_workflows` · `get_workflow` · `create_workflow` · `update_workflow` · `delete_workflow`\* · `duplicate_workflow` · `activate_workflow` · `deactivate_workflow` · `create_workflow_step` · `update_workflow_step` · `delete_workflow_step` · `reorder_workflow_step` · `add_subscriber_to_workflow` · `remove_subscriber_from_workflow`

Workflow V2 when enabled: `list_workflows_v2` · `get_workflow_v2` · `create_workflow_v2` · `configure_workflow_v2_draft` · `update_workflow_v2_draft` · `publish_workflow_v2`\* · `update_workflow_v2_status`\* · `add_subscriber_to_workflow_v2` · `remove_subscriber_from_workflow_v2` · `delete_workflow_v2`\*

#### Segments (5 tools)

`list_segments` · `get_segment` · `create_segment` · `update_segment` · `delete_segment`\* · `duplicate_segment`

#### Settings & metadata (6 tools)

`get_email_senders` · `get_custom_fields` · `get_email_snippets` · `get_writing_style` · `update_writing_style` · `update_organization_settings` · `get_available_variables`

#### Analytics (1 tool)

`get_org_stats`

#### Email (1 tool)

`send_email`\*

#### Web (1 tool)

`fetch_web_page`

#### Skill (1 tool)

`get_skill`

`*` = **dangerous** (requires double-confirmation, see below).

## Editing campaign content (the AI-agent path)

Email bodies are stored as **TipTap JSON documents**. The CLI's `campaigns update` only handles metadata; for the body itself, call `edit_campaign`:

### Mode 1 - replace the entire body

```bash
npx lumail tools run edit_campaign --params '{
  "id": "<campaignId>",
  "content": {
    "type": "doc",
    "content": [
      { "type": "paragraph", "content": [{ "type": "text", "text": "Hello!" }] },
      { "type": "spacer", "attrs": { "height": 16 } },
      { "type": "button", "attrs": { "text": "Click", "url": "https://example.com" } }
    ]
  }
}'
```

`content` accepts both a parsed object and a JSON-stringified value.

### Mode 2 - surgical operations (saves tokens, no full rewrite)

```bash
npx lumail tools run edit_campaign --params '{
  "id": "<campaignId>",
  "operations": [
    { "op": "replace_text", "search": "Hi there", "replace": "Hello friend", "all": true },
    { "op": "append_node", "node": { "type": "button", "attrs": { "text": "Subscribe", "url": "https://..." } } },
    { "op": "remove_node", "index": 3 }
  ]
}'
```

Operation types: `replace_text` · `insert_node` · `append_node` · `prepend_node` · `remove_node` · `replace_node`.

### Combining metadata + content/operations

```bash
npx lumail tools run edit_campaign --params '{
  "id": "<campaignId>",
  "subject": "New subject line",
  "preview": "New preheader",
  "operations": [{ "op": "replace_text", "search": "OLD", "replace": "NEW" }]
}'
```

> **Cannot combine `content` and `operations` in the same call** - pick one.

### Auto-fit image dimensions (editor magic wand)

Never guess image width and height. Use the image skill and the dedicated tool, which reads the actual remote image dimensions and applies the same maximum `600 × 400` sizing as the editor without upscaling or distorting the aspect ratio.

```bash
# Load the complete image workflow
lumail tools run get_skill --account <orgSlug> --params '{"type":"images"}' --json

# Inspect every nested image and proposed dimensions without writing
lumail tools run auto_fit_campaign_images --account <orgSlug> --params '{
  "campaignId": "<campaignId>",
  "dryRun": true
}' --json

# Apply all images, or pass imageIndexes from the dry-run response
lumail tools run auto_fit_campaign_images --account <orgSlug> --params '{
  "campaignId": "<campaignId>",
  "imageIndexes": [0, 2],
  "dryRun": false
}' --json
```

The result exposes each image's `src`, `alt`, TipTap `path`, natural dimensions, current dimensions, fitted dimensions, and application status so the model can reason from real metadata. The tool mutates draft broadcast campaigns or internal `WORKFLOW` emails, but keeps sent broadcast campaigns immutable. It never publishes, schedules, sends, or activates anything. Follow with `get_campaign` and `render_campaign`.

### TipTap node reference (cheat sheet)

| Node                                            | Minimal shape                                                                                                           |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| paragraph                                       | `{ "type": "paragraph", "content": [{ "type": "text", "text": "..." }] }`                                               |
| heading (level 2 or 3 only - **never 1**)       | `{ "type": "heading", "attrs": { "level": 2 }, "content": [...] }`                                                      |
| button                                          | `{ "type": "button", "attrs": { "text": "Click", "url": "https://..." } }`                                              |
| spacer (use instead of empty paragraphs)        | `{ "type": "spacer", "attrs": { "height": 16 } }`                                                                       |
| horizontalRule                                  | `{ "type": "horizontalRule" }`                                                                                          |
| image                                           | `{ "type": "image", "attrs": { "src": "url", "alt": "..." } }`                                                          |
| logo                                            | `{ "type": "logo", "attrs": { "src": "url", "size": "md" } }`                                                           |
| variable (call `get_available_variables` first) | `{ "type": "variable", "attrs": { "id": "name", "fallback": "there" } }`                                                |
| bulletList / orderedList                        | `{ "type": "bulletList", "content": [{ "type": "listItem", "content": [{ "type": "paragraph", "content": [...] }] }] }` |
| section (colored container)                     | `{ "type": "section", "attrs": { "backgroundColor": "#f7f7f7" }, "content": [...] }`                                    |
| columns                                         | `{ "type": "columns", "content": [{ "type": "column", "attrs": { "columnId": "a" }, "content": [...] }] }`              |
| footer                                          | `{ "type": "footer", "content": [{ "type": "text", "text": "..." }] }`                                                  |
| snippet (call `get_email_snippets` first)       | `{ "type": "snippet", "attrs": { "snippetId": "..." } }`                                                                |
| repeat (loop)                                   | `{ "type": "repeat", "attrs": { "each": "items" }, "content": [...] }`                                                  |

**Text marks** (in `content[].marks`): `bold`, `italic`, `underline`, `link` (`{ "type": "link", "attrs": { "href": "...", "target": "_blank" } }`), `textStyle` (`{ "type": "textStyle", "attrs": { "color": "#hex" } }`).

**Always include an unsubscribe link** before scheduling. The placeholder is `https://{{unsubscribeUrl}}`.

**Tag-action links** (let recipients add/remove tags by clicking): use `{{addTag:tag1,tag2}}` or `{{removeTag:tag1,tag2}}` as the `href`.

## Recipient targeting (filters & segments)

Campaigns must have recipients before being scheduled.

```bash
# 1. Discover available filter types
npx lumail tools run get_available_filters --params '{}'

# 2. Apply filters directly to a campaign (DRAFT only)
npx lumail tools run update_campaign_filters --params '{
  "campaignId": "<id>",
  "filters": [
    { "type": "TAG", "field": "tags", "operator": "BELONGS_TO_ANY", "tagIds": ["t1","t2"] },
    { "type": "STRING", "field": "email", "operator": "CONTAINS", "value": "@gmail.com" }
  ]
}'

# 3. Or create a reusable segment, then reference it
npx lumail tools run create_segment --params '{
  "name": "Active EU subscribers",
  "filters": [{ "type": "SYSTEM", "field": "system", "operator": "IS_INACTIVE" }]
}'
npx lumail tools run update_campaign_filters --params '{
  "campaignId": "<id>",
  "filters": [{ "type": "SEGMENT", "field": "segment", "operator": "INCLUDE", "segmentId": "...", "segmentName": "..." }]
}'
```

Filter types: `STRING` · `TAG` · `SEGMENT` · `NUMBER` · `CAMPAIGN` (interaction) · `WORKFLOW` (status) · `DATE` · `SYSTEM` (`IS_INACTIVE`/`IS_SUSPECT`) · `FIELD` (custom) · `CAPTURE_PAGE`. Filters combine with **AND**; for OR semantics, group via segments.

## Workflows (automation)

### Workflow V2: complete agent-built drafts

Workflow V2 is graph-based and versioned. When the URL contains `/workflows-v2/`, never use Workflow V1 tools and never call `create_campaign` for an email step.

Load `$lumail-workflow-v2` for the canonical mental model, all step and edge contracts, goals versus success goals, global exit rules versus terminal EXIT steps, complete payload examples, and the production verification checklist.

Use this exact sequence:

```bash
# Always target the organization named in /orgs/<orgSlug>/
lumail tools run get_skill --account <orgSlug> --params '{"type":"workflow_v2"}' --json

# Read the graph and preserve its exact updatedAt value
lumail tools run get_workflow_v2 --account <orgSlug> --params '{"workflowId":"<workflowId>"}' --json

# Inspect the live machine-readable contract when needed
lumail tools get configure_workflow_v2_draft --account <orgSlug> --json

# Configure the complete draft in one atomic call
lumail tools run configure_workflow_v2_draft --account <orgSlug> --params '<complete JSON>' --json
```

`configure_workflow_v2_draft` accepts the complete `steps` and `edges` arrays plus optional settings and success goals. It can create or update:

- internal `WORKFLOW` emails from inline subject, preview, sender, reply-to, and TipTap content;
- `WAIT`, `WAIT_UNTIL`, `CONDITION`, `SPLIT`, `MOVE_TO_STEP`, `ACTION`, `GOAL`, and `EXIT` steps;
- conversion goals, success goals, exit rules, repeat settings, and workflow groups.

For J0/J1/J3/J5 schedules, waits are incremental: 1 day from J0 to J1, then 2 days from J1 to J3, then 2 days from J3 to J5.

The call uses `expectedUpdatedAt` for optimistic concurrency, validates the complete graph, and returns the resolved definition plus `configuredEmails`. It never publishes and never sends. Publishing remains a separate confirmed action.

Keep the operation atomic. Never create temporary minimal workflow emails and enrich them later with campaign tools. If the call times out or rolls back, the draft is unchanged: call `get_workflow_v2` again, refresh `expectedUpdatedAt`, and retry the same complete payload. A timeout is a deployment/runtime issue, not a reason to leave a progressive partial graph behind.

If the installed CLI is stale or `bun` is not on `PATH`, run the current source CLI explicitly:

```bash
cd /Users/melvynx/Developer/projects/lumail.io
/Users/melvynx/.bun/bin/bun src/cli/index.ts tools get configure_workflow_v2_draft --account <orgSlug> --json
```

### Workflow V1: linear automations

```bash
# Create a workflow + first email step
npx lumail tools run create_workflow --params '{ "name": "Onboarding" }'
npx lumail tools run create_workflow_step --params '{
  "workflowId": "<wfId>",
  "type": "EMAIL",
  "name": "Welcome email"
}'
# An empty campaign is auto-created. Fill its body with edit_campaign:
npx lumail tools run edit_campaign --params '{
  "id": "<auto-created campaignId>",
  "content": { "type": "doc", "content": [...] }
}'

# Other step types
npx lumail tools run create_workflow_step --params '{ "workflowId": "<id>", "type": "WAIT", "config": { "duration": 86400 } }'
npx lumail tools run create_workflow_step --params '{ "workflowId": "<id>", "type": "ACTION", "config": { ... } }'
npx lumail tools run create_workflow_step --params '{ "workflowId": "<id>", "type": "WEBHOOK", "config": { ... } }'

# Activate
npx lumail tools run activate_workflow --params '{ "workflowId": "<id>" }'
```

Step types: `TRIGGER` (auto-added), `EMAIL`, `WAIT`, `ACTION`, `WEBHOOK`.

## Dangerous actions (double-confirmation)

These tools are flagged `dangerous` and require a Redis-backed challenge-response:

`schedule_campaign` · `cancel_campaign_schedule` · `send_email` · `unsubscribe` · `delete_tag` · `delete_segment` · `delete_workflow` · `publish_workflow_v2` · `update_workflow_v2_status` · `delete_workflow_v2`

Flow:

```bash
# 1. First call returns a 5-digit code
npx lumail tools run schedule_campaign --params '{ "campaignId": "<id>" }'
# -> { "error": "Confirmation required", "confirmationCode": 12345, "message": "..." }

# 2. Second call with the code executes the action
npx lumail tools run schedule_campaign --params '{
  "campaignId": "<id>",
  "confirmationCode": 12345
}'
```

Codes expire in ~60s and are bound to the exact tool + arguments.

## Common AI-agent recipes

### Create + populate + filter + schedule

```bash
# 1. Create draft
CID=$(npx lumail tools run create_campaign --params '{"subject":"Spring sale","name":"Spring 2026"}' --json | jq -r '.data.campaign.id')

# 2. Generate content
npx lumail tools run edit_campaign --params "{\"id\":\"$CID\",\"content\":{...}}"

# 3. Set recipients
npx lumail tools run update_campaign_filters --params "{\"campaignId\":\"$CID\",\"filters\":[{...}]}"

# 4. Send a test
npx lumail tools run send_test_email --params "{\"campaignId\":\"$CID\",\"emails\":[\"me@you.com\"]}"

# 5. Schedule (dangerous - 2 calls)
npx lumail tools run schedule_campaign --params "{\"campaignId\":\"$CID\",\"date\":\"2026-04-01\",\"hours\":10,\"timezone\":\"Europe/Paris\"}"
# -> grab the code, then:
npx lumail tools run schedule_campaign --params "{\"campaignId\":\"$CID\",\"date\":\"2026-04-01\",\"hours\":10,\"timezone\":\"Europe/Paris\",\"confirmationCode\":12345}"
```

### Repair a workflow auto-generated email

When a workflow EMAIL step is auto-generated, its campaign starts empty. To fill it, always use `edit_campaign` with `content` (not `operations`) for the first edit:

```bash
npx lumail tools run edit_campaign --params '{
  "id": "<auto-created campaignId>",
  "subject": "Welcome",
  "content": { "type": "doc", "content": [...] }
}'
```

## Gotchas

- **`campaigns update` is metadata-only.** For the body, use `tools run edit_campaign`.
- **Heading level 1 is rejected.** Use `level: 2` or `3`.
- **Empty paragraphs trigger warnings.** Use `spacer` nodes instead.
- **`content` and `operations` are mutually exclusive** in `edit_campaign`.
- **Variables must be nodes**, never plain text - `{ type: "variable", attrs: { id: "name" } }`, not `"@name"`.
- **`update_campaign_filters` only works on DRAFT** campaigns.
- **Dangerous actions cannot be retried** with a stale `confirmationCode` (60s TTL).
- **An unsubscribe link is required** before `schedule_campaign` succeeds.
- **`required: true` variables must have a non-empty `fallback`.**

## Output Format

CLI output (without `--json`) is a human-readable table. With `--json`:

```json
{ "ok": true, "data": { ... } }
```

On error: `{ "ok": false, "error": { "message": "...", "status": 400 } }`.

V2 tool calls (`tools run ...`) wrap the result in `{ "success": true, "data": ... }` and validation errors include `issues[]` with field paths and human-readable hints.

## Quick Reference

```bash
npx lumail --help                       # Top-level commands and global flags
npx lumail <resource> --help            # Actions for a resource
npx lumail <resource> <action> --help   # Flags for one action
npx lumail tools list                   # Every tool enabled for the organization
npx lumail tools get <tool-name>        # Inspect tool schema
```

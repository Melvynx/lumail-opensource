---
name: lumail-plugin
version: "0.4.0"
tags: ["lumail", "email", "marketing", "newsletter", "mcp", "oauth", "plugin"]
description: How to use the Lumail MCP connection installed by the Lumail plugin. Read this before the first Lumail MCP tool call in a conversation, and whenever a Lumail tool errors with an authorization or "tool not found" problem.
---

# Lumail plugin — MCP basics

The Lumail plugin connects Claude Code and Codex to one Lumail organization over
an OAuth MCP connection at `https://lumail.io/mcp`. There is no API token to
create, paste, or store: the host runs the browser OAuth flow, the user picks the
organization, and the access token stays in the host's credential store.

## Before the first tool call

Confirm the server is connected and authenticated.

Claude Code:

```bash
claude mcp get plugin:lumail:lumail
```

Codex:

```bash
codex mcp list
```

Expected: status connected, URL `https://lumail.io/mcp`. If the status is
"needs authentication", run the login command for the host —
`claude mcp login plugin:lumail:lumail` or `codex mcp login lumail` — and tell
the user a browser window will open. Do not ask the user for an API token; this
connection does not use one.

## What the connection can and cannot do

The OAuth connection exposes a deliberately narrowed tool set. It is scoped to
the single organization the user picked during consent, and to that user's
membership — only owners and admins can authorize it.

**Read** (`lumail.read` scope): `list_subscribers`, `get_subscriber`,
`list_tags`, `list_campaigns`, `get_campaign`, `render_campaign`,
`get_campaign_analytics`, `get_campaign_progress`, `list_workflows`,
`get_workflow`, `list_workflow_groups`, `get_workflow_group`, `get_org_stats`,
`get_subscriber_stats`, `get_subscriber_growth`, `get_email_senders`,
`get_custom_fields`, `get_email_snippets`, `get_writing_style`,
`get_available_variables`, `get_skill`.

**Write** (`lumail.write` scope): `add_subscriber`, `get_or_create_tags`,
`bulk_add_tags`, `bulk_remove_tags`, `create_campaign`, `edit_campaign`,
`duplicate_campaign`, `auto_fit_campaign_images`, `create_workflow`,
`configure_workflow_draft`, `update_workflow_draft`, `create_workflow_group`,
`update_workflow_group`, `set_workflow_group`.

**Not available over OAuth at all**: sending a campaign, scheduling a send,
publishing or activating a workflow, unsubscribing someone, and every delete.
These are not permission-gated — the tools are not registered on this endpoint.
If the user asks for one, say so plainly and point them at the Lumail app, or at
the API-token MCP endpoint if they want the full catalog in an agent.

## Working rules

- **Read before you write.** Call the matching `list_*` or `get_*` tool before
  creating or editing anything, so you edit the record the user means.
- **Campaigns start as drafts.** `create_campaign` and `edit_campaign` produce
  drafts. Never imply a campaign was sent — it was not, and it cannot be from
  here.
- **Workflows are configured as one graph.** Call `get_skill` with type
  `workflow` first, read the current state with `get_workflow`, then call
  `configure_workflow_draft` passing the exact `updatedAt` value you read. A
  stale `updatedAt` is rejected, which is the intended optimistic lock.
- **Tags are resolved, not invented.** `get_or_create_tags` returns the ids that
  `bulk_add_tags` and `bulk_remove_tags` expect.
- **Bulk tag operations are real and immediate.** Confirm the target set with
  `list_subscribers` and repeat the count back to the user before running one.

## Rate limits

Per organization, per minute: 100 on Free, 700 on Premium, 2,000 on Business. A
`429` carries `Retry-After`; wait it out rather than retrying immediately.

## Errors

- **401 with a `WWW-Authenticate` header** — the access token expired or was
  revoked. Re-run the host's login command.
- **"Only Lumail organization owners and admins can authorize the plugin"** —
  the user is a member, not an owner or admin, of the org they selected. They
  need to re-authorize picking an org where they are owner or admin.
- **"Tool not found"** — either the tool is one of the send/delete tools that
  this endpoint intentionally omits, or it is gated off for the organization's
  plan. Check `list_*` availability before assuming a bug.

## When to use the CLI instead

The `lumail` skill covers `npx lumail`, which reaches the full API surface
including sending. It authenticates separately with `npx lumail auth login`
(same browser OAuth, `lumail.cli` scope). Use it when the user explicitly wants
an action this MCP connection does not expose, and only after they ask for it.

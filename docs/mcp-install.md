# Lumail MCP install instructions — any MCP client

This connects an arbitrary MCP client to Lumail. Prefer the dedicated guides
when one exists for the client's host: Claude Code (lumail.io/claude-code),
Codex (lumail.io/codex), or ChatGPT (lumail.io/chatgpt). This guide is for
everything else — Cursor, Windsurf, custom agents, and scripts.

## Agent execution contract

If you are an agent reading this on a user's behalf, read the whole guide
before changing configuration. Edit the client's config yourself, then verify.
Do not stop after writing the config: the connection is useless until the user
has authenticated.

## 1. Environment gate

MCP configuration lives on a local machine. If this conversation is running in
a web workspace, a cloud sandbox, or a CI runner, stop and tell the user to run
their agent on their own machine with this prompt:

`Read lumail.io/mcp/install and connect this MCP client to Lumail.`

## 2. Pick the connection style

**A. OAuth (preferred).** `https://lumail.io/mcp` is an OAuth 2.1 protected
resource with dynamic client registration and PKCE. Clients that support OAuth
MCP register themselves and open a browser window: the user signs in, **chooses
which organization the client may reach**, and approves the scopes
(`lumail.read`, `lumail.write`). Only owners and admins can authorize an
organization. No token is ever shown or stored in the config file.

**B. API token (legacy, full access).** Clients without OAuth support can use
`https://lumail.io/api/mcp/sse` with an `Authorization: Bearer lum_...`
header. The user creates the token in Lumail under **Settings > API Tokens**.
Warn the user plainly: this endpoint exposes the **full** tool set, including
sending and deletes, and the token in the config file is a secret — do not
commit it.

Default to A. Only fall back to B when the client cannot do OAuth.

## 3. Configure the client

Adapt to the client you are running in. Common shapes:

Cursor (`.cursor/mcp.json` or Settings > MCP):

```json
{
  "mcpServers": {
    "lumail": {
      "url": "https://lumail.io/mcp"
    }
  }
}
```

Codex-style TOML (`~/.codex/config.toml`):

```toml
[mcp_servers.lumail]
url = "https://lumail.io/mcp"
oauth_resource = "https://lumail.io/mcp"
```

Clients without HTTP MCP support can bridge through stdio:

```json
{
  "mcpServers": {
    "lumail": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://lumail.io/mcp"]
    }
  }
}
```

Never invent a different URL. The OAuth endpoint is exactly
`https://lumail.io/mcp`, without `/api` or `/sse`; the token endpoint is
exactly `https://lumail.io/api/mcp/sse`.

## 4. Authenticate and verify

Trigger the client's MCP login or reconnect flow, walk the user through the
browser step, then verify by listing the server's tools. On the OAuth endpoint
you must see read and draft-write tools (subscribers, tags, campaigns,
workflows, analytics) and you must **not** see any send, publish, unsubscribe,
or delete tool — their absence is the safety feature, not a bug.

## What the connection can do

Read: subscribers, tags, campaigns and their analytics, campaign send progress,
workflows and workflow groups, org stats, subscriber growth, senders, custom
fields, snippets, writing style.

Write: add subscribers, create and apply tags in bulk, create and edit campaign
drafts, duplicate campaigns, create and configure workflow drafts.

**Sending, scheduling, publishing or activating a workflow, unsubscribing, and
every delete are not exposed over this connection at all.** Those tools are not
registered on the OAuth endpoint, so an agent cannot reach them even with the
user's approval. That work happens in the Lumail app, or with `npx lumail` (see
the `lumail` skill). Say this plainly rather than letting a user believe an
agent sent something.

Rate limits are per organization: 100 requests/minute on Free, 700 on Premium,
2,000 on Business.

## Removal

Delete the `lumail` entry from the client's MCP configuration, or revoke the
authorization from the connected-apps list in Lumail settings — revoking there
kills access immediately without touching local config.

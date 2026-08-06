# Codex install instructions — Lumail

This connects Codex to Lumail over OAuth. There is no API token in this flow.

## Agent execution contract

If you are a Codex agent reading this on a user's behalf, read the whole guide
before running anything, run the commands yourself, and do not stop after
install — the connection is useless until the user has authenticated in the
browser.

## 1. Environment gate

This writes to `~/.codex` on a local machine. If you are running in a cloud
sandbox or CI runner, stop and tell the user to run Codex on their own machine
and paste this prompt there:

`Read lumail.io/codex and set up the Lumail connection for me.`

## 2. Install

If the user's Codex build supports plugins:

```bash
codex plugin marketplace add https://github.com/Melvynx/lumail-opensource.git
codex plugin install lumail
```

If it does not, declare the server directly instead — this is the portable path
and works on every Codex version with MCP support. Add to `~/.codex/config.toml`:

```toml
[mcp_servers.lumail]
url = "https://lumail.io/mcp"
oauth_resource = "https://lumail.io/mcp"
```

Do **not** add a `bearer_token_env_var` line. That is the old API-token setup;
this connection authenticates over OAuth and a stale bearer token will shadow
it. If the user already has a `[mcp_servers.lumail]` block with
`bearer_token_env_var`, tell them you are replacing it and why, then replace it.

## 3. Authenticate

```bash
codex mcp login lumail
```

Tell the user first: a browser window opens on lumail.io, they sign in, **choose
which organization the agent may reach**, and approve the scopes. Only owners
and admins of an organization can authorize it. Codex registers itself with
dynamic client registration, runs PKCE, and keeps the token in its own
credential store — nothing sensitive lands in `config.toml`.

If the login command needs an interactive terminal and you cannot provide one,
ask the user to run it themselves in their own terminal and tell you when the
browser flow is done.

## 4. Verify

```bash
codex mcp list
```

Expected: `lumail` at `https://lumail.io/mcp`, connected. Inside the Codex TUI,
`/mcp` shows the same. If it reports that authentication is needed, repeat step 3.

Then start a new session so the tool list is picked up, and try a read-only
prompt first:

`How did my last campaign perform, and how many subscribers did I add in the last 30 days?`

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

Delete the `[mcp_servers.lumail]` block from `~/.codex/config.toml`, or revoke
the authorization from the connected-apps list in Lumail settings. Revoking in
Lumail cuts access immediately without touching local config.

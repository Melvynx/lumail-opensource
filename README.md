# Lumail Agent Skills and Plugins

Install Lumail's agent skills or connect [Lumail](https://lumail.io) to Claude
Code and Codex. The repository includes reusable marketing and copywriting
skills alongside the Lumail CLI and OAuth MCP plugin skills.

Authentication is browser OAuth. **There is no API token to create, paste, or
store.**

## Install

The fastest path is to let your agent do it. Paste this into Claude Code:

```text
Read lumail.io/claude-code and install the Lumail plugin for me.
```

or into Codex:

```text
Read lumail.io/codex and set up the Lumail connection for me.
```

or, for the terminal workflow (full API, including sending), into any agent:

```text
Read lumail.io/cli and set up the Lumail CLI for me.
```

Prefer to run it yourself:

```bash
claude plugin marketplace add https://github.com/Melvynx/lumail-skills.git
claude plugin install lumail@lumail
claude mcp login plugin:lumail:lumail
```

Full guides: [docs/claude-code-install.md](./docs/claude-code-install.md) ·
[docs/codex-install.md](./docs/codex-install.md)

## What is in here

| Path                                | What it is                                                     |
| ----------------------------------- | -------------------------------------------------------------- |
| `.claude-plugin/marketplace.json`   | Claude Code marketplace descriptor                             |
| `claude/.claude-plugin/plugin.json` | Claude Code plugin, declares the MCP server inline             |
| `codex/.codex-plugin/plugin.json`   | Codex plugin metadata                                          |
| `codex/.mcp.json`                   | Codex MCP server configuration                                 |
| `.agents/plugins/marketplace.json`  | Codex marketplace descriptor                                   |
| `skills/lumail/`                    | The `lumail` CLI skill (`npx lumail`) — canonical copy         |
| `skills/lumail-plugin/`             | How to drive the MCP connection, its limits and errors         |
| `skills/marketing/`                 | Marketing strategy, positioning, offers, funnels, and launches |
| `skills/copywritting/`              | Conversion copy for emails, pages, ads, scripts, and CTAs      |

`claude/skills` and `codex/skills` are real copies of `skills/` (symlinks break
on Windows checkouts), so both hosts ship the same skills. Edit the
top-level `skills/` first, then sync the copies.

## How the connection works

The plugin points both hosts at `https://lumail.io/mcp`, an OAuth 2.1 protected
resource. The host registers itself through dynamic client registration, runs
PKCE, and stores the access token itself. During the browser flow you sign in,
pick **which organization** the agent may reach, and approve the scopes. Only
owners and admins of an organization can authorize it.

Access tokens carry the organization as a claim, so a session can never reach
another organization in your account. Revoke from the connected-apps list in
Lumail settings and access stops on the next request, with no local config to
clean up.

## What agents can do

**Read** — subscribers, tags, campaigns, campaign analytics and progress,
workflows and workflow groups, org stats, subscriber growth, senders, custom
fields, snippets, writing style.

**Write** — add subscribers, create and apply tags, create and edit campaign
drafts, duplicate campaigns, create and configure workflow drafts.

**Not available over OAuth** — sending, scheduling, publishing or activating a
workflow, unsubscribing, and every delete. Those tools are not registered on
this endpoint, so an agent cannot reach them even with your approval. Do that
work in the Lumail app, or with `npx lumail` if you want it in a terminal.

## The CLI

`npx lumail` reaches the full API, including sending. It authenticates with the
same browser flow:

```bash
npx lumail auth login
```

The `lumail` skill in this repository teaches agents the whole CLI.

## Example prompts

- `How did last week's newsletter perform compared to the one before?`
- `Draft a launch campaign with three subject line options.`
- `Add user@example.com to my newsletter and tag them vip.`
- `Build a three-email welcome sequence over a week and leave it as a draft.`
- `Show me subscriber growth over the last 30 days.`

## Support

[lumail.io](https://lumail.io) · [Documentation](https://lumail.io/docs/ai-integration)

MIT licensed.

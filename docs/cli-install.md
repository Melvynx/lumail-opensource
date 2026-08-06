# Lumail CLI install instructions

This sets up the Lumail CLI and the `lumail` skill, so an agent can drive the
full email-marketing API from the terminal — including the operations the OAuth
plugin deliberately does not expose: sending, scheduling, publishing workflows,
unsubscribing, and deletes.

## Agent execution contract

If you are an agent reading this on a user's behalf, read the whole guide
before running anything. Run the commands yourself — do not merely describe
them. Do not stop after install: the setup is useless until the user has
authenticated and you have verified a read command.

Required order:

1. Environment gate.
2. Install the CLI.
3. Authenticate over OAuth in the browser.
4. Install the `lumail` skill.
5. Verify.
6. Print the required self-report.

## 1. Environment gate

The CLI and skill install onto a local machine. If this conversation is running
in a web workspace, a cloud sandbox, or a CI runner, stop here — that
environment cannot hold the user's credential. Tell the user to open their
agent on their own machine and paste this prompt:

`Read lumail.io/cli and set up the Lumail CLI for me.`

If you are unsure which environment you are in, ask before running commands.

## 2. Install the CLI

The CLI is published on npm as `lumail`. Install it globally with the package
manager the user already uses — check for pnpm, bun, then fall back to npm:

```bash
npm install -g lumail
lumail --version
```

If the user prefers no global install, prefix every command below with
`npx lumail` instead of `lumail` and say so in your self-report.

## 3. Authenticate

```bash
lumail auth login
```

Tell the user before you run it: a browser window opens on lumail.io, they sign
in, **choose which organization the CLI may reach**, and approve the access.
Only owners and admins of an organization can authorize it. The credential
lands in `~/.config/lumail/oauth.json` (mode 0600) and refreshes itself —
there is no API token to create, paste, or rotate.

**Do not ask the user for an API token.** `auth login` is the default flow.
API tokens (`lum_...`) still exist for CI and named accounts via
`lumail auth set`, but do not push the user toward one.

`auth login` needs to open a browser and wait on a loopback callback. If you
cannot provide an interactive session, ask the user to run
`lumail auth login` themselves in their own terminal and tell you when the
browser flow is done.

## 4. Install the lumail skill

The skill teaches the agent every CLI command, the output formats, and the
safety rules. Install it into the skills directory of the agent you are running
in — detect which directories exist and install into each one that applies:

- Claude Code: `~/.claude/skills/lumail/SKILL.md`
- Codex and other agents reading a shared skills directory:
  `~/.agents/skills/lumail/SKILL.md`

```bash
curl -fsSL --create-dirs \
  -o ~/.claude/skills/lumail/SKILL.md \
  https://raw.githubusercontent.com/Melvynx/lumail-opensource/main/skills/lumail/SKILL.md
```

If the `skills` CLI is available, it does the same thing in one command:

```bash
pnpm dlx skills add Melvynx/lumail-opensource -g
```

If the user has the Lumail **plugin** installed (Claude Code or Codex), the
skill already ships with it — skip this step and say so.

## 5. Verify

```bash
lumail auth test
lumail campaigns list --json
```

`auth test` must succeed, and the campaigns call must return JSON (an empty
list is fine on a new organization). If either fails with an authentication
error, repeat step 3 before continuing.

## 6. Required self-report

Print one short report before ending your turn:

- The CLI version installed and how it authenticates (OAuth, which
  organization).
- Where the skill was installed (which directories).
- The verification result.

Do not tell the user setup is complete before `auth test` and a real read
command have both passed.

## Safety contract — read this before your first write

Unlike the OAuth plugin, the CLI reaches the **full** API: `campaigns send`,
`emails send`, unsubscribes, and every delete are all available. Never run a
send, publish, unsubscribe, or delete without the user's explicit confirmation
in the current conversation. Default to drafts and dry runs, and quote the
exact command back before executing anything irreversible.

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

## Update and removal

```bash
npm update -g lumail
```

To disconnect: `lumail auth remove` clears the local credential, and
revoking the app from Lumail settings kills access immediately. Delete the
skill directory (`~/.claude/skills/lumail` or `~/.agents/skills/lumail`) to
remove the skill.

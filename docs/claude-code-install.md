# Claude Code install instructions — Lumail

This installs the Lumail plugin from the public marketplace repository:

https://github.com/Melvynx/lumail-opensource.git

## Agent execution contract

If you are a Claude Code agent reading this on a user's behalf, read the whole
guide before running anything. Run the commands yourself — do not merely
describe them. Do not stop after install: the connection is useless until the
user has authenticated, and this conversation cannot use the tools it installs.

Required order:

1. Environment gate.
2. Install the marketplace and the plugin.
3. Authenticate over OAuth in the browser.
4. Verify.
5. Hand off to a new conversation, and print the required self-report.

## 1. Environment gate

The plugin installs into `~/.claude` on a local machine. If this conversation is
running in a web workspace, a cloud sandbox, or a CI runner, stop here — that
environment cannot modify the user's local plugin directory. Tell the user to
open Claude Code on their own machine and paste this prompt into a new
conversation:

`Read lumail.io/claude-code and install the Lumail plugin for me.`

If you are unsure which environment you are in, ask before running commands.

## 2. Install

Check the CLI is present and recent enough — plugin marketplaces need Claude
Code 2.x:

```bash
claude --version
```

If `claude` is not on `PATH`, look for the CLI bundled with the Claude desktop
app before installing anything new. On macOS it lives under
`~/Library/Application Support/Claude/claude-code/<version>/claude.app/Contents/MacOS/claude`
(pick the newest `<version>`). Use that full quoted path in place of `claude`
everywhere below.

```bash
claude plugin marketplace add https://github.com/Melvynx/lumail-opensource.git
claude plugin install lumail@lumail
```

The marketplace registers as `lumail`. If the install name is rejected, run
`claude plugin marketplace list` and use the exact name it reports.

## 3. Authenticate

**There is no API token in this flow.** Do not ask the user for one, do not read
`~/.config/lumail/token`, and do not add an `Authorization` header anywhere. The
plugin points at `https://lumail.io/mcp`, an OAuth 2.1 resource with dynamic
client registration; Claude Code registers itself, runs PKCE, and stores the
resulting token in its own credential store.

```bash
claude mcp login plugin:lumail:lumail
```

Tell the user before you run it: a browser window opens on lumail.io, they sign
in, **choose which organization the agent may reach**, and approve the requested
scopes. Only owners and admins of an organization can authorize it.

`claude mcp login` needs an interactive terminal. If it fails with "stdin isn't
a terminal" — the usual outcome when an agent runs it through a shell tool —
run it under a pseudo-TTY instead:

```bash
CLAUDE_BIN="${CLAUDE_BIN:-claude}" python3 - <<'PY'
import os, pty, select, signal, sys, time
claude = os.environ.get("CLAUDE_BIN", "claude")
pid, fd = pty.fork()
if pid == 0:
    os.execvp(claude, [claude, "mcp", "login", "plugin:lumail:lumail"])
end = time.time() + 180
status = None
while time.time() < end:
    r, _, _ = select.select([fd], [], [], 1)
    if r:
        try:
            data = os.read(fd, 4096)
        except OSError:
            break
        if not data:
            break
        os.write(1, data)
    done, code = os.waitpid(pid, os.WNOHANG)
    if done:
        status = code
        break
if status is None:
    os.kill(pid, signal.SIGTERM)
    _, status = os.waitpid(pid, 0)
sys.exit(os.waitstatus_to_exitcode(status))
PY
```

Set `CLAUDE_BIN` to the full CLI path when `claude` is not on `PATH`. This
wrapper is macOS/Linux only; on Windows, ask the user to run
`claude mcp login plugin:lumail:lumail` themselves in an interactive PowerShell
window. Wait for the authenticated confirmation in the output, and treat a
non-zero exit as a failed login.

## 4. Verify

```bash
claude mcp get plugin:lumail:lumail
```

Expected: type `http`, URL `https://lumail.io/mcp`, and a connected status. If
it reports "needs authentication", repeat step 3. Note that
`claude plugin details lumail@lumail` reports "MCP servers (0)" even on a
correct install — that is a display quirk of that command, not a failure, so
trust `claude mcp get` instead.

## 5. Required final step: hand off to a new conversation

Claude Code captures a session's MCP tool list at session start, so **this
conversation cannot call the Lumail tools it just installed.** Do not try; the
calls will fail. A new conversation is mandatory.

Use the user's own conversation language for the startup prompt. English:

`The Lumail plugin is installed. Show me an overview of my email operation — recent campaigns and how they performed, subscriber growth over the last 30 days, and my tags — then tell me what you can do from here: drafting campaigns, adding and tagging subscribers, and building workflow sequences. Then ask me what I want to work on.`

Do these in order:

1. **Handoff chip.** If a task-spawning tool is available in this session (for
   example `spawn_task` on the Claude Code desktop session server), call it once
   with title "Lumail overview" and the startup prompt above as the new
   session's prompt. Tell the user to click the chip to begin.
2. **Fallback.** If no such tool exists or the call errors, print the startup
   prompt in a copyable block and tell the user to open a new Claude Code
   conversation and paste it there.

Print exactly one self-report before ending your turn:

- **(A)** "Created a one-click handoff — click the chip to start." Report (A)
  only if the spawn call actually succeeded. Explain that this installation
  conversation cannot use Lumail and the new one is where the work happens.
- **(B)** "Could not create the handoff automatically" — give the paste-in
  prompt and say which tool was unavailable or which call failed.

Do not tell the user setup is complete until you have printed (A) or (B). Never
claim you started a new conversation — the user always triggers it.

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
claude plugin marketplace update lumail
claude plugin update lumail@lumail
```

Restart Claude Code afterwards, then re-run the verification command.

To disconnect, remove the plugin with `claude plugin uninstall lumail@lumail`,
or revoke the authorization from the connected-apps list in Lumail settings —
revoking there kills access immediately without touching local config.

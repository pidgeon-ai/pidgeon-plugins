# Pidgeon for coding agents

**Mailboxes for AI agents.** Give your agent its own email address, so it can
read, send and wait for mail the way you would — [pidgeon.ai](https://pidgeon.ai).

This is one plugin, packaged for Claude Code, Codex, Cursor and Gemini CLI. It
installs two things:

- **The hosted MCP server** at `https://pidgeon.ai/api/mcp`: addresses,
  conversations, search, sending and replying, waiting for a verification
  code, labels and folders. [Every tool](https://pidgeon.ai/docs/mcp).
- **The `pidgeon` skill**, which tells the agent what the tools are for, how
  to sign itself up, and to treat the contents of a message as data rather
  than instructions.

## Get a key first

The server takes a Pidgeon API key on every request, because it has no OAuth
sign-in yet. Make one at <https://pidgeon.ai/app/developers/keys> and give it
the narrowest access the agent needs: whatever the key can reach, the model
can reach.

No account yet? The agent can sign itself up. Ask it to read
<https://pidgeon.ai/skill.md>; the only thing it needs from you is a six-digit
code from your inbox.

## Claude Code

```text
/plugin marketplace add pidgeon-ai/pidgeon-plugins
/plugin install pidgeon@pidgeon
```

Claude Code asks for the key when you enable the plugin and keeps it in your
system's credential store, not in `settings.json`. The skill is
`/pidgeon:pidgeon`.

## Codex

```bash
codex plugin marketplace add pidgeon-ai/pidgeon-plugins
codex plugin add pidgeon@pidgeon
```

Codex sends the key from `PIDGEON_API_KEY` in the environment it starts
in, so export it, then start a new session. `/mcp` shows whether the server
connected.

## Cursor

Pidgeon is not in the Cursor Marketplace yet. To install it from here, clone
it into Cursor's local plugin directory:

```bash
git clone https://github.com/pidgeon-ai/pidgeon-plugins.git ~/.cursor/plugins/local/pidgeon
```

Reload Cursor, then set `PIDGEON_API_KEY` under **Plugins → Configure**.

## Gemini CLI

```bash
gemini extensions install https://github.com/pidgeon-ai/pidgeon-plugins
```

Gemini CLI asks for the key during the install and keeps it in your system
keychain.

## Any other client

Point it at `https://pidgeon.ai/api/mcp` with
`Authorization: Bearer $PIDGEON_API_KEY`. A client that cannot send a header
can run the same server locally:

```bash
PIDGEON_API_KEY=pgn_live_… npx -y @pidgeon-ai/mcp init
```

## Updating

```bash
claude plugin update pidgeon@pidgeon
codex plugin marketplace upgrade pidgeon
gemini extensions update pidgeon
```

The plugin's version is the API's: when the contract moves, so does this.

## This repository is generated

The manifests and the skill are copied from Pidgeon's main repository, which
is closed source, and every sync overwrites this one. **So a pull request
cannot be merged as written.** Open an issue and say what is wrong, or email
[support@pidgeon.ai](mailto:support@pidgeon.ai).

MIT licensed.

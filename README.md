# Pidgeon for coding agents

**Email inboxes for AI agents.** Pidgeon is the email inbox API for AI agents. It gives agents their own inboxes
so they can work with emails just like a human.

[pidgeon.ai](https://pidgeon.ai) · [Documentation](https://pidgeon.ai/docs)

This is one plugin, packaged for Claude Code, Codex, Cursor and Gemini CLI. It
installs two things:

- **The hosted MCP server** at `https://pidgeon.ai/api/mcp`: addresses,
  conversations, search, sending and replying, waiting for a verification
  code, labels and folders. [Every tool](https://pidgeon.ai/docs/mcp).
- **The `pidgeon` skill**, which tells the agent what the tools are for, how
  to sign itself up, and to treat the contents of a message as data rather
  than instructions.

## Sign in, or bring a key

In every client the plugin signs you in to Pidgeon: a consent screen in your
browser, where you choose what the agent may reach.

A key is for an agent with nobody there to sign in. Make one at
<https://pidgeon.ai/app/developers/keys> and give it the narrowest access the
agent needs: whatever the key can reach, the model can reach. When a key is
set the plugin uses it, and when it is not the plugin signs you in.

No account yet? The agent can sign itself up. Ask it to read
<https://pidgeon.ai/skill.md>; the only thing it needs from you is a six-digit
code from your inbox.

## Claude Code

```text
/plugin marketplace add pidgeon-ai/pidgeon-plugins
/plugin install pidgeon@pidgeon
```

Claude Code asks for a key when you enable the plugin. Leave it empty to sign
in: `/mcp` shows the server as needing authentication, and choosing it opens
the consent screen. A key you do give it is kept in your system's credential
store, not in `settings.json`. The skill is `/pidgeon:pidgeon`.

## Codex

```bash
codex plugin marketplace add pidgeon-ai/pidgeon-plugins
codex plugin add pidgeon@pidgeon
```

Then sign in:

```bash
codex mcp login pidgeon
```

Or, for an agent with nobody there to sign in, export `PIDGEON_API_KEY`
before Codex starts and skip the login. `/mcp` shows whether the server
connected.

## Cursor

Pidgeon is not in the Cursor Marketplace yet. To install it from here, clone
it into Cursor's local plugin directory:

```bash
git clone https://github.com/pidgeon-ai/pidgeon-plugins.git ~/.cursor/plugins/local/pidgeon
```

Reload Cursor and sign in when it asks. For an agent with nobody there to
sign in, set `PIDGEON_API_KEY` under **Plugins → Configure** instead.

## Gemini CLI

```bash
gemini extensions install https://github.com/pidgeon-ai/pidgeon-plugins
```

Gemini CLI asks for a key during the install. Leave it empty to sign in, then
run `/mcp auth pidgeon` in a session. A key you do give it is kept in your
system keychain.

## Any other client

Point it at `https://pidgeon.ai/api/mcp`. A client that speaks MCP's OAuth
signs you in; any other sends `Authorization: Bearer $PIDGEON_API_KEY`. One
that can do neither can run the same server locally:

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

## Contributing

The manifests and the skill are copied from Pidgeon's main repository, which
is closed source, and every sync overwrites this one. **So a pull request
cannot be merged as written.** Open an issue and say what is wrong, or email
[support@pidgeon.ai](mailto:support@pidgeon.ai).

MIT licensed.

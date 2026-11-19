# Pidgeon for coding agents

**Identity and inboxes for AI agents.** Pidgeon gives each agent a verified
identity of its own: a key on your machine, signed requests, and a reputation
the websites it visits can check. Each agent can have its own inbox too.

[pidgeon.ai](https://pidgeon.ai) · [Documentation](https://pidgeon.ai/docs)

This is one plugin, packaged for Claude Code, Codex, Cursor and Gemini CLI. It
installs three things:

- **The hosted MCP server** at `https://pidgeon.ai/mcp`: registering agents
  and making their keys, reputation, inboxes and mail, drafts held for your
  approval, and for websites, checking the agents that arrive.
  [The MCP server](https://pidgeon.ai/docs/sdks#mcp).
- **The `pidgeon` skill**, which tells the agent what the tools are for, that
  an agent's private key never leaves your machine, and to treat the
  contents of a message as data rather than instructions.
- **The `pidgeon-bouncer` skill**, which adds Pidgeon Bouncer to a Next.js
  app's proxy, so the site sees which agents call it and who runs them.
  It is a standalone Agent Skill too, for v0 and any agent that reads them:
  `npx skills add pidgeon-ai/pidgeon-plugins --skill pidgeon-bouncer`.
  [Bouncer in v0](https://pidgeon.ai/docs/bouncer#v0).

## Sign in, or bring a key

In every client the plugin signs you in to Pidgeon: a consent screen in your
browser, where you choose what the agent may reach.

A key is for an agent with nobody there to sign in. Make one at
<https://pidgeon.ai/keys> and give it the narrowest access the
agent needs: whatever the key can reach, the model can reach. When a key is
set the plugin uses it, and when it is not the plugin signs you in.

No account yet? Sign in at <https://pidgeon.ai/sign-in> with your email or
Google; the plugin's first call takes you there too.

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

Point it at `https://pidgeon.ai/mcp`. A client that speaks MCP's OAuth
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

## Contributing

Open an issue and say what is wrong, or email
[support@pidgeon.ai](mailto:support@pidgeon.ai).

MIT licensed.

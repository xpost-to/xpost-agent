<p align="center">
  <img src="assets/logo.svg" alt="xpost" width="96" />
</p>

# xpost for your assistant

Post to X, Instagram, LinkedIn, Facebook, TikTok, YouTube, Threads, Bluesky
and Pinterest from Claude, Cursor, Gemini CLI or any MCP client. Every post
lands in **your approval queue** first — no key, tool or prompt can approve
one. Brand guardrails run before anything is queued, and each account gets
its own delivery receipt.

This repo is the open-source **skill** (the workflow an agent should
follow) plus the plugin manifests that install it together with the hosted
**MCP server** at `https://xpost.to/api/mcp`. You sign in once in a browser
window; nothing to paste.

You need an xpost account — the first post is free, no card:
https://xpost.to/signup

## Install

### Any agent — as a skill

```bash
npx skills add xpost-to/xpost-agent
```

This installs the skill only. Add the MCP server in your client as below,
or let the skill's REST fallback use `XPOST_API_KEY`.

### Claude Code — plugin (skill + MCP server)

```
/plugin marketplace add xpost-to/xpost-agent
/plugin install xpost@xpost-agent
```

Then `/mcp` → **xpost** → sign in. Or without the plugin:

```bash
claude mcp add --transport http xpost https://xpost.to/api/mcp
```

### Gemini CLI — extension (skill + MCP server)

```bash
gemini extensions install https://github.com/xpost-to/xpost-agent
```

Gemini opens a browser to sign in to xpost on first use; `/mcp auth xpost`
signs in again.

### Cursor

One click: [Add xpost to Cursor](cursor://anysphere.cursor-deeplink/mcp/install?name=xpost&config=eyJ0eXBlIjoiaHR0cCIsInVybCI6Imh0dHBzOi8veHBvc3QudG8vYXBpL21jcCJ9)

Or add it by hand in **Settings → Tools & Integrations → New MCP Server**:

```json
{ "mcpServers": { "xpost": { "type": "http", "url": "https://xpost.to/api/mcp" } } }
```

Then `npx skills add xpost-to/xpost-agent` for the skill. The repo also
carries a Cursor plugin manifest (`.cursor-plugin/plugin.json`).

### ChatGPT, Claude.ai, Claude Desktop

ChatGPT: xpost is in the plugin directory — find it, add it, sign in.
Claude: **Settings → Connectors → Add custom connector**, paste
`https://xpost.to/api/mcp`, sign in.

### No MCP client at all — the CLI

```bash
npx xpost login
npx xpost posts create -c "Shipped it." -a x:yourhandle
```

The `xpost` package on npm is the CLI, the local MCP server (`npx xpost mcp`)
and the skill's fallback in one. Docs: https://xpost.to/docs/cli

## What the skill teaches

- Accounts first: `list_accounts` names every destination and what it can carry.
- `get_posting_rules` is the option catalog per account; a key that is not there cannot be used.
- A held post (`pending_approval`) is the normal outcome, not an error. Say so; print the preview link.
- A guardrail refusal names the rule. Rewrite once. Never evade.
- `list_posts` before assuming a post is still waiting — the person may have rejected it with a reason.
- Receipts are per account: "delivered to X, failed on Instagram: reason".

Full text: [`skills/xpost/SKILL.md`](skills/xpost/SKILL.md).

## Docs

- Every MCP tool: https://xpost.to/docs/mcp
- REST API (OpenAPI): https://xpost.to/api/v1/openapi.json
- Set-up guides per client: https://xpost.to/docs

## Self-hosting a client against your own instance

Replace `https://xpost.to` with your instance in the MCP address. For the
REST fallback set `XPOST_URL` and `XPOST_API_KEY`.

## License

MIT. The skill and manifests here are free to copy and adapt; xpost itself
is a hosted service.

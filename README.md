# Postguard agent skill

Give any Claude-family agent the ability to post through Postguard — with the
approval queue and guardrails enforced server-side, so the skill can't be
prompt-injected into bypassing them.

## Install (Claude Code)

Copy the skill into your project (or `~/.claude/skills/` for all projects):

```bash
cp -r skill/postguard /path/to/your-project/.claude/skills/postguard
```

Then either register the MCP server (preferred):

```bash
claude mcp add postguard -e POSTGUARD_API_KEY=pg_live_... \
  -e POSTGUARD_URL=http://localhost:3001 \
  -- node /path/to/postguard/mcp/server.mjs
```

…or export `POSTGUARD_URL` + `POSTGUARD_API_KEY` for the REST fallback the
skill documents. Create the key in Postguard → Dashboard → API keys (agent
key, so copilot approval applies).

When this repo goes public, the skill will be installable directly from
GitHub (e.g. `npx skills add <org>/postguard-skill`).

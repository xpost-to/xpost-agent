# xpost agent skill

Give any Claude-family agent the ability to post through xpost — with the
approval queue and guardrails enforced server-side, so the skill can't be
prompt-injected into bypassing them.

## Install (Claude Code)

Copy the skill into your project (or `~/.claude/skills/` for all projects):

```bash
cp -r skill/xpost /path/to/your-project/.claude/skills/xpost
```

Then either register the MCP server (preferred):

```bash
claude mcp add xpost -e XPOST_API_KEY=xp_live_... \
  -e XPOST_URL=http://localhost:3001 \
  -- node /path/to/xpost/mcp/server.mjs
```

…or export `XPOST_URL` + `XPOST_API_KEY` for the REST fallback the
skill documents. Create the key in xpost → Dashboard → API keys (agent
key, so copilot approval applies).

When this repo goes public, the skill will be installable directly from
GitHub (e.g. `npx skills add <org>/xpost-skill`).

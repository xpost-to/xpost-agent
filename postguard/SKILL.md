---
name: postguard
description: Post to social media through Postguard — draft, schedule, and check delivery across X, Instagram, Bluesky, TikTok, YouTube, LinkedIn, Facebook, Threads, and Pinterest, with human approval and brand guardrails enforced. Use when asked to post, schedule, or cross-post social content, or to check whether a post was delivered.
---

# Posting through Postguard

Postguard is the posting layer between you and the user's social audience.
Every post you create passes brand guardrails, and in copilot mode it is held
for the user's approval before anything publishes. This is by design — never
try to work around it.

## Connection

Preferred: the `postguard` MCP server (tools: `list_accounts`, `create_post`,
`list_posts`, `get_delivery_receipt`). If MCP is not available, use the REST
API directly with the environment variables `POSTGUARD_URL` (default
`http://localhost:3001`) and `POSTGUARD_API_KEY`:

```bash
# List connected accounts (get ids for posting)
curl -s "$POSTGUARD_URL/api/v1/social-accounts" \
  -H "Authorization: Bearer $POSTGUARD_API_KEY"

# Create a post (omit scheduled_at to post as soon as it's approved)
curl -s -X POST "$POSTGUARD_URL/api/v1/posts" \
  -H "Authorization: Bearer $POSTGUARD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"caption": "...", "social_accounts": ["<account-uuid>"],
       "scheduled_at": "2026-08-01T17:00:00Z"}'

# Delivery receipt (per-platform status, live URL, or error detail)
curl -s "$POSTGUARD_URL/api/v1/post-results?post_id=<post-uuid>" \
  -H "Authorization: Bearer $POSTGUARD_API_KEY"
```

## Workflow

1. **Attach media when needed** — call `upload_media` with a local file `path` or a public `url` to get a media_id. Instagram/TikTok/YouTube/Pinterest REQUIRE media. For a video you can pick the cover frame by passing `thumbnail_timestamp_ms` (ms into the video) when uploading via the REST API.
2. **List accounts first** — you need account ids, and the platforms tell you
   the constraints (e.g. Instagram/TikTok/YouTube/Pinterest require media).
3. **Create the post.** Per-platform caption overrides go in
   `platform_configurations`, e.g.
   `{"x": {"caption": "shorter version for X"}}`. To post a story or reel
   instead of a feed post, set `placement` there:
   `{"instagram": {"placement": "stories"}}` — instagram/facebook take
   `reels`/`stories`/`timeline`, threads takes `reels`/`timeline`. Reels
   require a video; stories require media. Invalid combinations are rejected
   at create time with a clear error.
4. **Read the response status honestly:**
   - `pending_approval` — expected in copilot mode. Tell the user their
     approval is needed (dashboard or Telegram); do NOT retry or treat it as
     an error.
   - `422 Blocked by workspace guardrails` — the caption violated a brand
     rule; the violations array says which. Rewrite the caption to comply and
     try once more. Never attempt to evade a guardrail.
   - `scheduled` / `posted` — done; report the scheduled time.
5. **Check the delivery receipt** after publishing time and report the live
   post URL, or the per-platform error if delivery failed.

## Rules

- One post per ask — don't fan out variations unless the user asked.
- Respect platform limits you know (X: 280 chars; Instagram: media required).
- When the user gives no schedule, prefer `scheduled_at` omitted (post now /
  after approval) and say so.
- Report outcomes faithfully, including partial failures ("delivered to X,
  failed on Instagram: <reason>").

---
name: xpost
description: Post to social media through xpost — draft, schedule, and check delivery across X, Instagram, Bluesky, TikTok, YouTube, LinkedIn, Facebook, Threads, and Pinterest, with human approval and brand guardrails enforced. Use when asked to post, schedule, or cross-post social content, or to check whether a post was delivered.
---

# Posting through xpost

xpost is the posting layer between you and the user's social audience.
Every post you create passes brand guardrails, and in copilot mode it is held
for the user's approval before anything publishes. This is by design — never
try to work around it.

## Connection

Preferred: the `xpost` MCP server (tools: `list_accounts`, `create_post`,
`bulk_post`, `list_posts`, `get_delivery_receipt`, `check_guardrails`,
`upload_media`, plus the learning loop: `get_post_metrics`, `get_insights`,
`get_top_posts`, `get_best_times`). Two ways to connect: the hosted
Streamable HTTP connector at `<instance>/api/mcp` (Bearer = the API key; or
`<instance>/api/mcp/<key>` for clients that can't set headers), or the local
stdio server (`node mcp/server.mjs`) for localhost instances and host-file
uploads. If MCP is not available, use the REST
API directly with the environment variables `XPOST_URL` (default
`http://localhost:3001`) and `XPOST_API_KEY` — full spec at
`$XPOST_URL/api/v1/openapi.json`:

```bash
# List connected accounts (get ids for posting)
curl -s "$XPOST_URL/api/v1/social-accounts" \
  -H "Authorization: Bearer $XPOST_API_KEY"

# Create a post (omit scheduled_at to post as soon as it's approved)
curl -s -X POST "$XPOST_URL/api/v1/posts" \
  -H "Authorization: Bearer $XPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"caption": "...", "social_accounts": ["<account-uuid>"],
       "scheduled_at": "2026-08-01T17:00:00Z"}'

# Delivery receipt (per-platform status, live URL, or error detail)
curl -s "$XPOST_URL/api/v1/post-results?post_id=<post-uuid>" \
  -H "Authorization: Bearer $XPOST_API_KEY"

# Bulk (up to 100 rows) — JSON rows, or POST a CSV with
# header caption,accounts[,scheduled_at,media_urls] ("|" separates
# multi-values inside a cell). Every row runs the normal guardrail +
# approval path; the response reports each row separately.
curl -s -X POST "$XPOST_URL/api/v1/posts/bulk" \
  -H "Authorization: Bearer $XPOST_API_KEY" \
  -H "Content-Type: text/csv" \
  --data-binary @posts.csv
```

## Workflow

1. **Attach media when needed** — call `upload_media` with a local file `path` or a public `url` to get a media_id. Instagram/TikTok/YouTube/Pinterest REQUIRE media. For a video you can pick the cover frame by passing `thumbnail_timestamp_ms` (ms into the video) when uploading via the REST API. A **PDF** is a LinkedIn document post (each page a swipeable slide): one PDF, attached on its own, LinkedIn only — set the LinkedIn option `document_title` to name it, or it takes the file's name.
2. **List accounts first** — you need account ids, and the platforms tell you
   the constraints (e.g. Instagram/TikTok/YouTube/Pinterest require media).
3. **Create the post.** Per-platform caption overrides go in
   `platform_configurations`, e.g.
   `{"x": {"caption": "shorter version for X"}}`. To post a story or reel
   instead of a feed post, set `placement` there:
   `{"instagram": {"placement": "stories"}}` — instagram/facebook take
   `reels`/`stories`/`timeline`, threads takes `reels`/`timeline`. Reels
   require a video; stories require media. Invalid combinations are rejected
   at create time with a clear error. For X, `{"x": {"first_comment": "…"}}`
   posts that text as a reply right under the tweet (the link-in-first-comment
   pattern), and `{"x": {"thread": ["tweet 2", "tweet 3"]}}` (up to 4
   follow-ups) posts a chained thread under the main tweet. Instagram takes a
   first comment too — `{"instagram": {"first_comment": "…"}}`, 2200 characters,
   not on stories — but only for accounts connected through bundle.social;
   anywhere else Instagram refuses to post the comment and the create call
   fails, naming the account. Threads takes `topic_tag` (one word), a
   `reply_control` audience, a poll (`poll_option_a` … `poll_option_d`, at
   least two, and only on a post with no media — the API is the only way to
   set one; the dashboard has no poll fields), a `gif_id` (a giphy.com link
   — also media-free) and `crosspost_ig_story`. Where the poster has been paid
   or the content is AI-made, the platforms' own labels are options rather than
   something to write in the caption: `is_paid_partnership` (+
   `branded_content_sponsors`) and `is_ai_generated` on Instagram,
   `is_brand_content` / `is_organic_brand_content` / `is_ai_generated` on
   TikTok, `has_paid_product_placement` / `contains_synthetic_media` on
   YouTube, `is_ai_generated` on Pinterest. When uploading an
   image you can pass `alt_text` (accessibility description — applied on X
   and Bluesky). Every text field — caption, overrides, first comment, thread
   tweets — goes through the workspace guardrails; hiding content in an
   override is rejected the same as in the caption.
4. **Read the response status honestly:**
   - `pending_approval` — expected in copilot mode. Tell the user their
     approval is needed (dashboard or Telegram); do NOT retry or treat it as
     an error.
   - `422 Blocked by project guardrails` — the caption violated a brand
     rule; the violations array says which. Rewrite the caption to comply and
     try once more. Never attempt to evade a guardrail.
   - `scheduled` / `posted` — done; report the scheduled time.
5. **Check the delivery receipt** after publishing time and report the live
   post URL, or the per-platform error if delivery failed.

## Bulk posting

For a batch (a content calendar, a CSV the user hands you), use `bulk_post`
(or `POST /api/v1/posts/bulk`) instead of looping `create_post`. Account
references may be account ids, `platform:username`, or a bare username when
it's unique. Rows are independent — ALWAYS relay the per-row report: which
rows were created (and their status — `pending_approval` in copilot mode is
normal for the whole batch), and which failed with what reason. Never retry
the whole batch when only some rows failed; fix and resend just those rows.

## Learning what works

Before drafting, when the account has history: `get_top_posts` shows which
past posts actually performed (study angle, length, format — don't copy
captions verbatim), and `get_best_times` ranks posting slots from the
workspace's own results (few samples = weak signal; say so). After a post has
been live a while, `get_post_metrics` gives per-platform engagement — null
means the platform doesn't report that metric, not zero, and empty data means
metrics haven't synced yet (they refresh ~6h). Use `get_insights` for
"how are we doing" summaries and platform comparisons.

## Rules

- One post per ask — don't fan out variations unless the user asked.
- Respect platform limits you know (X: 280 chars; Instagram: media required).
- When the user gives no schedule, prefer `scheduled_at` omitted (post now /
  after approval) and say so.
- Report outcomes faithfully, including partial failures ("delivered to X,
  failed on Instagram: <reason>").

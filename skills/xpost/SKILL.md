---
name: xpost
description: Post to social media through xpost — draft, schedule, edit and check delivery across X, Instagram, LinkedIn, Facebook, TikTok, YouTube, Threads, Bluesky and Pinterest, with their brand rules enforced server-side and their approval where the project asks for it. Use whenever asked to post, schedule, cross-post or take down social content, or to check whether a post went out and how it did.
---

# Posting through xpost

xpost is the posting layer between you and the user's social audience. It
holds their connected accounts, their brand rules and, where they have asked
for one, their approval queue. **Creating a post sends it**: it publishes at
the time you gave it, or straight away if you gave none — so create one only
when the person has asked for it, and show them what it says first. A project
in copilot mode holds your posts for approval instead (`get_project` says
which); never try to work around that queue where it exists.

A post made by hand in a browser is invisible to all three, and cannot be
tracked, retried or taken down from here. When the user asks to post,
use these tools.

## Connection

**Preferred: the `xpost` MCP server.** If it is connected you have the
tools below and nothing else is needed. If it is not connected, tell the
user how — no key required:

- Claude Code: `claude mcp add --transport http xpost https://xpost.to/api/mcp`, then `/mcp` → xpost → sign in.
- Cursor / Windsurf / Gemini CLI / VS Code: add `https://xpost.to/api/mcp` as an HTTP MCP server; the client opens the sign-in page.
- ChatGPT: xpost is in the plugin directory — find it, add it, sign in.
- Claude.ai / Claude Desktop: Settings → Connectors → Add custom connector, paste `https://xpost.to/api/mcp`, sign in.

**Fallback: the `xpost` CLI** (`npx xpost`, zero install), when there is no
MCP client — a shell, a script, a headless box. Every command is one route of
the public API and answers with the API's own JSON; a refusal prints the
body and exits 1, so read `error_code`. Sign in once with `npx xpost login`
(a browser opens), or set `XPOST_API_KEY` where no browser can. The tool
names below map one to one:

```bash
npx xpost project                      # get_project
npx xpost accounts                     # list_accounts
npx xpost rules <accountId>            # get_posting_rules
npx xpost upload ./pic.jpg --alt "…"   # upload_media → media id
npx xpost posts create -c "…" -a <account> [-a …] [--media <id>] [--at 2026-10-01T09:00:00+02:00] [--draft] [--config '{"x":{"first_comment":"…"}}']
npx xpost posts list --status pending_approval,rejected
npx xpost posts edit <postId> -c "…"   # update_post — answers with a NEW id
npx xpost posts delete <postId>
npx xpost receipt <postId>             # get_delivery_receipt
```

Raw HTTP works too: `XPOST_URL` (default `https://xpost.to`) +
`Authorization: Bearer $XPOST_API_KEY`, spec at `$XPOST_URL/api/v1/openapi.json`.

## The tools

| Do this | Tool |
|---|---|
| Learn the project: timezone, approval mode, signature, guardrails, allowance | `get_project` |
| See destinations and what each can carry | `list_accounts` |
| The exact options an account takes (placements, first comment, boards…) | `get_posting_rules` |
| Attach a file the user gave you (path, URL, or a chat attachment) | `upload_media` — or `create_post` with `files` / `media_pending` |
| Create one post | `create_post` |
| Create many | `bulk_post` |
| Find posts, see verdicts and rejection reasons | `list_posts` |
| Change words, media, time or options | `update_post` |
| Remove your own unpublished post | `delete_post` |
| Where it landed, per account | `get_delivery_receipt` |
| A destination failed — try it again | `retry_delivery` |
| Pull a live post down (only when asked) | `take_down_post` |
| Which accounts need reconnecting | `list_connection_issues` |
| Test a caption against the rules before writing more | `check_guardrails` |
| Learn what works | `get_post_metrics`, `get_insights`, `get_top_posts`, `get_best_times` |

## Workflow

1. **`get_project`, then `list_accounts`.** You need account ids, and each
   account says what it can carry (`limits`, `can`). An **empty** list comes
   with `connect_message` and `connect_url`: nothing can be posted until the
   person connects an account, and you may be the only one who can tell them.
   Pass both on, verbatim.
2. **`get_posting_rules`** for the accounts you will use, whenever you set
   anything beyond a caption. It is the option catalog per ACCOUNT — a key
   that is absent cannot be used on that account, whatever the platform
   allows elsewhere. Pinterest answers live `boards` with `eligible`; pin
   only to eligible ones.
3. **Media.** Instagram, TikTok, YouTube and Pinterest require it; stories
   and reels do too. `upload_media` takes a local `path` (stdio server only),
   a public `url`, or base64 `data`. A picture, video or PDF the user
   attached to the chat goes ON THE POST, two ways:
   - Your host hands attachments over as file objects (ChatGPT does): pass
     them as `files` on `create_post` or `update_post`, in post order.
   - Otherwise create the post with `media_pending: true`. It is held, and
     the reply carries `upload_url` and `upload_curl`: if the file is in
     your environment, run that command (YOUR_FILE replaced) and the picture
     lands on the post — nothing passes through a tool argument, so no host
     asks anyone to allow an opaque payload. Where you cannot run it, the
     approval card that came back carries a drop zone: the person adds the
     picture there, straight onto the post, and approves in the same card.
     Either way nothing comes back to you; the post simply has its picture.
     To check, `list_posts` shows `mediaIds`. Base64 `data` on
     `upload_media` is for something you generated, not the user's file.
   "I've added another photo" therefore needs nothing from you — it is
   already on the post. To add one from the chat, `update_post` with `files`
   or `add_media_ids`; `media_ids` replaces the set.
   A PDF is a LinkedIn document post: one PDF, alone, LinkedIn only.
4. **`create_post`.** `scheduled_at` is ISO 8601 with an offset, in the
   project's timezone (from `get_project`); omit it and the post goes out
   now (or, in copilot mode, as soon as they approve it). **`"next_slot"`** hands the time to the project's own posting
   schedule — use it when the person says "queue it", or names no time and
   `get_project` shows `queue.configured: true` (its `next_slots` are what
   they will get). Refused as `no_queue_slots` where there is no schedule;
   then ask for a time rather than inventing one. `is_draft: true` files a scratchpad draft — no queue, no
   approval link; the person sends it from its page. Per-account words and
   options go in `platform_configurations`, keyed by platform:
   `{"x": {"caption": "shorter", "first_comment": "…", "thread": ["…"]}}`,
   `{"instagram": {"placement": "stories"}}`. Every text field — caption,
   overrides, first comment, thread — runs through the guardrails.
5. **Read the answer honestly.**
   - `scheduled` — the normal outcome: nothing is holding it. Say when it
     goes out, in the project's timezone, and print the **See post preview**
     link the result carries, first, before any summary — it is how they see
     the post, and change or cancel it while there is still time.
   - `pending_approval` — this project holds agent posts. Say the post is
     waiting for their approval and print the same link first. Do not retry.
     Do not say "published".
   - `ok: false` with `error_code` — a refusal, as data. Guardrail
     violations name the rule in `violations[].detail`: rewrite once to
     comply, never evade. A missing plan (`publishing_needs_plan`) or a
     missing account comes with a `message` written to be passed on.
6. **Before assuming a post is still waiting, `list_posts`.** Its `from` /
   `to` bound the scheduled time, so "what goes out next week" is one call,
   soonest first. The person can
   reject through a link you never see; a `rejected` row carries
   `rejectionReason`, which is what to fix. To change anything, `update_post`
   — it answers with a **new id**, and an edit to an approved post goes back
   into the queue because the approval was for the old words.
7. **After publish time, `get_delivery_receipt`.** One row per destination:
   report the live link, or the reason it failed, per account. `retry_delivery`
   for a failed row; `take_down_post` only when the user asks for it, because
   there is no undo.

## Bulk

For a calendar or a CSV, `bulk_post` (≤100 rows) instead of a loop. Rows are
independent — always relay the per-row report: which were created (every row
`scheduled`, or `pending_approval` on a project that holds agent posts, is
normal) and which failed, with the reason. Resend only the failed rows. Give each row an `idempotency_key` so a
resend after a timeout returns `repeated: true` instead of a second copy.

## Learning what works

Before drafting, when there is history: `get_top_posts` shows what actually
performed (study angle, length and format; never copy captions), and
`get_best_times` ranks slots from this workspace's own results — read
`signal` first: `none` or `thin` means there is no history to speak of, so
say so rather than name a best hour. After a post has been live a while,
`get_post_metrics`; `null` means the platform does not report that metric,
not zero, and stories are never reported on by anyone. `get_insights` for
"how are we doing".

## Rules

- One post per ask. Do not fan out variations unless asked.
- Never promise a time you did not get back. Timing comes from
  `scheduled_at` and `publishes` in the answer, on the project's clock.
- A guardrail block means rewrite once; a held post means tell the person.
- Report outcomes faithfully, including partial ones: "delivered to X,
  failed on Instagram: <reason>".
- Upload plumbing (sandboxes, hosts, slots) is not something the person can
  act on. The card says what it needs; your words go on what is still open,
  like a missing caption.

---
name: mellow-hub
description: Find the short videos taking off on TikTok, Instagram and YouTube, then publish or schedule one post to Instagram, TikTok, YouTube, X, LinkedIn, Threads, Bluesky, Pinterest, Facebook and Telegram through Mellow Hub (remote MCP server or REST API), and run Instagram comment-to-DM auto-replies. Use whenever asked to publish to social networks, cross-post, schedule posts, check a post against a network's rules, find trending videos, or answer Instagram comments and messages via Mellow Hub.
metadata:
  version: "1.1.0"
  homepage: https://www.mellow.world/hub/guides/social-media-mcp
  mcp: https://www.mellow.world/mcp
---

# Mellow Hub: from trend to post, through an agent

Mellow Hub finds what is taking off, publishes one post to many social
networks and checks every network's own rules before anything goes out. Ten
networks: Instagram, TikTok, YouTube, X, LinkedIn, Threads, Bluesky,
Pinterest, Facebook and Telegram (TikTok connects two ways: a personal account,
or TikTok Business). You reach it two ways, and both resolve to the same
functions:

- **MCP** (preferred): `https://www.mellow.world/mcp`, Streamable HTTP, OAuth
  2.1 with dynamic client registration and PKCE, or an owner-issued API key as
  the bearer. An OAuth connection is served 39 tools; a credential holding
  `ai:generate` also gets the Creative Studio generation tools and
  `trends_breakdown` (53 in all). 4 guide resources (`mellow://guide/start`,
  `platforms`, `errors`, `recipes`) and 3 prompts (`publish_everywhere`,
  `weekly_plan`, `repurpose_video`).
- **REST**: `https://www.mellow.world/api/hub/v1`, `Authorization: Bearer
  mk_live_…` (an API key the owner creates at `https://www.mellow.world/hub/agents`).

Full reference: https://www.mellow.world/hub/docs (the same text as
`https://www.mellow.world/hub/llms.txt` and the MCP resource `mellow://guide/start`).
Setup, per client and per network: https://www.mellow.world/hub/guides/social-media-mcp

## 1. Connect

### MCP with OAuth (Claude Code, Claude.ai, ChatGPT, Cursor, Hermes, OpenClaw…)

Claude Code:

```sh
claude mcp add --transport http mellow https://www.mellow.world/mcp
```

Then authenticate when the client asks. The server answers the first request
with a 401 that points to `/.well-known/oauth-protected-resource` and names
every scope except `ai:generate`; a client that supports OAuth discovery
completes registration, PKCE and the consent screen by itself. The **person**
decides on the consent screen:

- which channels (all are ticked at first; they can narrow it to some);
- which scopes (what the client asked for is ticked, except `ai:generate`,
  which is always opt-in and then carries a daily ceiling of AI credits);
- review or autopilot (autopilot is preselected when the client asked to
  publish);
- a daily ceiling of posts (1 to 1,000, 50 at first);
- how long the access lasts (until revoked at first; or 90, 30, 7 or 1 days).

Generic `mcp.json` (Cursor, Codex, OpenClaw and other clients that read this
shape):

```json
{
  "mcpServers": {
    "mellow": {
      "type": "http",
      "url": "https://www.mellow.world/mcp"
    }
  }
}
```

If the client cannot do OAuth, an API key works as the bearer on the same
endpoint (never paste it into chat; read it from the environment):

```json
{
  "mcpServers": {
    "mellow": {
      "type": "http",
      "url": "https://www.mellow.world/mcp",
      "headers": { "Authorization": "Bearer ${MELLOW_HUB_KEY}" }
    }
  }
}
```

Hermes: `hermes mcp add --url https://www.mellow.world/mcp --auth oauth mellow`
then `hermes mcp login mellow`.

A client keeps the tool list it loaded. When `whoami` lists `newTools`, a tool
was added after this connection was made: if it is missing from your list, ask
the person to reconnect or refresh the Mellow Hub connector.

### REST with an API key

Only a signed-in person creates keys, at `/hub/agents` (`POST /api/hub/v1/keys`
takes `name`, `scopes`, `mode`, `channelScope`, `dailyPostLimit` (50 when left
out), `dailyCreditLimit` (only with `ai:generate`) and `expiresInDays`). A key's
scopes, when left out, are `channels:read media:write posts:read posts:write
posts:publish metrics:read`; `channels:connect` and `ai:generate` must be added
on purpose. Keys start with `mk_live_` and are bound to one profile; a profile
header cannot switch them.

```sh
curl --fail-with-body https://www.mellow.world/api/hub/v1/whoami \
  -H "Authorization: Bearer $MELLOW_HUB_KEY"
```

## 2. The required call order

Always in this order. Each step's answer changes what the next may do.

| Step | MCP tool | REST | What to read |
| --- | --- | --- | --- |
| 1 | `whoami` | `GET /whoami` | `mode` (`autopilot` publishes; `review` only prepares), `scopes`, the `channels` this credential may touch (with `needsReconnect` or `signInEndsAt` where a sign-in has run out or is about to), `postsRemainingToday`, `subscription.active` and `subscription.upgradeUrl` (over REST: `actor.mode`, `actor.scopes`, `limits.remainingToday`). If `active` is false, say so to the person before doing anything else: publishing will be refused, everything else works. |
| 2 | `list_channels` | `GET /channels` | Channel ids: `spc_…` for most networks, `tgc_…` for a Telegram channel. These are what you name in `channels`. Pass `refresh: true` only after a person just connected something. |
| 3 | `register_media`, `request_upload_url` or the Library | `POST /media`, `POST /media/upload-url` | For a public https URL: `register_media` checks it and returns `kind`/`contentType`; nothing is copied, the network fetches the URL at publish time, so it must still resolve then. For bytes you hold, or any video: `request_upload_url`, PUT the bytes to `uploadUrl` with their `Content-Type`, then use `mediaUrl`. For the person's own photos, see section 8. |
| 4 | `validate_post` | `POST /validate` | Publishes nothing, spends nothing, returns **every** problem at once: `{ ok, issues[], notes[] }`. Fix all `issues`, re-validate, and only then continue. Read `notes` too: they say what Hub fitted for a network (`adapted_*`) and what may go wrong before it does (section 6). |
| 5 | `create_post` | `POST /posts` with `Idempotency-Key` header (or `idempotencyKey` in the body) | Publishes now, or schedules with `scheduledAt`. Under review mode it prepares the post (`pending_review`) and stops. Never call without validating first. |
| 6 | `get_post` | `GET /posts/{id}` | Read `targets`, not just `status`. Each channel succeeds or fails on its own; `partial` means some published. A successful create is not proof every network published. |

Afterwards, as needed: `preview_post` (how it will read per network),
`list_posts` (find a post after a timeout), `update_post` (change a post that
has not gone out, keeping its id), `reschedule_post`, `retry_post`,
`cancel_post` (needs `confirm: true`), `post_results` (how one post did, per
network), `account_overview` (the profile at a glance), `get_metrics` (needs
`metrics:read`), `mark_post_finished` (a post the person put up by hand),
`read_audit_log` (this credential's own actions), `list_platforms` (needs no
scope).

## 3. Idempotency keys: non-negotiable

- `create_post` refuses a request without `idempotencyKey`: 8 to 120
  characters, stable for this post.
- Derive it from the task, not at random: `week-38-tue-morning`,
  `blog-142-linkedin`, `client-acme-launch-2026-09-25`.
- A timeout is not a refusal. Retry the **same** request with the **same**
  key: Mellow returns the original post instead of a second publication.
- Reusing a key for **different** content is refused with
  `idempotency_key_reused` (409). Do not generate a new key to get past it;
  compare with what you sent before.
- To change a post that has not gone out, call `update_post` with the whole
  post. Never create a second one.
- A `partial` post is mended with `retry_post` and the post's id: it sends the
  post again to the failed channels alone. Never create the post again.

## 4. The post

```json
{
  "caption": "The long version, written for LinkedIn…",
  "media": ["https://cdn.example.com/clip.mp4"],
  "channels": ["spc_linkedin", "spc_bluesky", "spc_x", "spc_youtube", "tgc_channel"],
  "scheduledAt": "2026-12-01T10:00:00Z",
  "options": {
    "youtube": { "title": "Why we rebuilt the editor", "privacyStatus": "public" }
  },
  "perChannel": {
    "spc_bluesky": { "caption": "The short version." },
    "spc_x": { "caption": "The short version." }
  },
  "idempotencyKey": "editor-rebuild-2026-12-01"
}
```

- `caption`: one text for the post; each network's limit is checked per
  channel.
- `media`: ordered list, up to 35 items; each a public https URL, or
  `{ url, thumbnailUrl?, thumbnailTimestampMs?, tags? }`, or
  `{ libraryAssetId }` for a picture from the person's Library. `tags` mark
  accounts on the picture (Instagram and Facebook only), not collaborators.
- `channels`: 1 to 25 channel ids, no duplicates.
- `scheduledAt`: ISO 8601 with a timezone, in the future; omit to publish now.
- `options`: per-network settings keyed by platform id; applies to every
  channel of that network.
- `perChannel`: overrides for one channel (caption, media, that network's
  options); wins over `options`. This is how Bluesky gets 300 characters while
  LinkedIn keeps 3,000.
- `draft: true`: keep it in Mellow, send nothing.

What you do not fit, Hub fits per channel and says so in the notes
(`adapted_*`): a caption over a limit ends at its last whole sentence that
fits, closing hashtags kept; a network that takes one file gets the first; a
YouTube title comes from the caption's first line; a Pinterest video gets its
first frame as cover. What would have to be invented (a video for YouTube,
media for Pinterest) is still an issue.

Platform ids: `instagram`, `tiktok`, `tiktok_business`, `youtube`, `x`,
`linkedin`, `threads`, `bluesky`, `pinterest`, `facebook`, `telegram`.

## 5. Per-network rules (from Mellow's platform registry)

| Network | Caption | Title | Media | Placement | Options worth knowing |
| --- | --- | --- | --- | --- | --- |
| Instagram | ≤ 2,200, optional | — | 1–10, image or video; stories exactly 1; reels 1 video | `timeline`, `reels`, `stories` | `placement`, `collaborators`, `shareToFeed`, `location`, `audioName`, `trialReelType` (`manual`/`performance`; one video as a reel only), `ordinaryIfTrialRefused` (off unless set: without it a refused trial fails and nothing is posted), `finishInApp` with `soundUrl` (the person finishes it in the Instagram app, for a library sound). Professional (Business or Creator) account only; connect with `connectionType` `instagram` (default) or `facebook`. One Instagram per profile. |
| TikTok | ≤ 2,200, optional | ≤ 90, optional (headline on photo posts) | 1 video **or** up to 35 images; never mixed | — | `title`, `privacyStatus` (`public`/`private`), `allowComment`, `allowDuet`, `allowStitch`, `discloseYourBrand`, `discloseBrandedContent`, `isAiGenerated`, `isDraft`, `autoAddMusic`. TikTok reviews every video before others see it, so a TikTok reads as sent rather than live. An account whose region is Russia cannot publish through an app: use `isDraft` and the person posts from the phone. |
| TikTok Business | ≤ 2,200, optional | ≤ 90, optional | 1 video or up to 35 images | — | Same as TikTok, plus a video thumbnail. A second way of connecting TikTok, not a network of its own. |
| YouTube | ≤ 5,000, optional (becomes the description) | ≤ 100, **required** | exactly 1 video (landscape allowed; square or vertical up to 3 minutes may become Shorts; there is no Shorts switch) | — | `title` (required), `description`, `tags`, `categoryId`, `defaultLanguage`, `privacyStatus` (`public`/`private`/`unlisted`), `embeddable`, `license`, `publicStatsViewable`, `publishAt`, `madeForKids`, `containsSyntheticMedia`, `recordingDate`, `localizations` |
| X | ≤ 280, optional (standard tier; Mellow cannot see premium; a link counts 23) | — | 0–4 images or 1 video; text alone is fine | — | `poll` (2–4 options, 5–10,080 minutes), `communityId`, `quoteTweetId`, `replySettings` |
| LinkedIn | ≤ 3,000, **required** | — | 0–20 images or 1 video | — | `resharePostId`. Posts as a personal profile or as a company page you administer (`connectionType` `personal` or `organization`). |
| Threads | ≤ 500, optional (an emoji counts its size in UTF-8) | — | 0–20; text alone is fine | `timeline`, `reels` | `placement`, `thread`: a chain of up to 24 replies after the post, each ≤ 500 with at most one picture or video. A chain needs the profile's own Threads sign-in ("Enable chains"); `list_channels` says `chains: ready` or `needs_connection`. |
| Bluesky | ≤ 300 **graphemes**, **required** | — | 0–4 images or 1 video | — | none. Connected on Mellow Hub's Channels page with a handle and an app password, never in the chat. |
| Pinterest | ≤ 500, optional (pin description) | ≤ 100, optional | exactly 1 image or video; a video needs a cover (`thumbnailUrl`, or Hub takes its first frame) | — | `title`, `boardIds`, `link`. Business account with at least one board. |
| Facebook | ≤ 63,206, optional | — | 0–10; stories exactly 1; reels 1 video | `timeline`, `reels`, `stories` | `placement`, `location`, `collaborators`, `setCaptionForEachImage`. Pages only, never personal timelines. |
| Telegram | ≤ 4,096, optional (up to 1,024 sit under the media; the rest follows as its own message) | — | 0–10 as an album; a video up to 50 MB, a picture up to 10 MB; text alone is fine | — | `silent`, `disableLinkPreview`. Telegram's styles from marks (`**bold**`, `__italic__`, `~~struck~~`, `\|\|hidden\|\|`, `` `code` ``, `[words](https://link)`, `> ` quote). A channel the person administers, connected by adding Mellow's bot inside Telegram. |

Video thumbnails: `thumbnailUrl` on the media item where the network accepts
one (Instagram, TikTok Business, YouTube, Facebook; a Pinterest cover). Hub
reads an MP4's or MOV's length, frame and weight from its header and notes
`media_outside_limit` when one is outside a limit the network's API
publishes; codecs and the network's processing remain the network's
judgement. Hub does not change a file.

## 6. Errors, notes and what to do

Every error carries `code`, a sentence and `retryable`. If `retryable` is
false, the same request will fail the same way; change the request or stop.

| Code | Where | What to do |
| --- | --- | --- |
| `subscription_required` / `plan_required` (402) | publishing, AI writing | Stop. Relay the message and `subscription.upgradeUrl` from `whoami` to the person. Connecting, drafting, validating and previewing keep working. |
| `monthly_limit_reached` (402) | `create_post` | The month's publishing ceiling is used (a post counts once per network it reaches). Publish to fewer networks or wait for the month; never split a post to get under it without asking. |
| `daily_limit_reached` (429) | `create_post`, moving a post | This credential's posts for that UTC day are used up (a post counts on the day it goes out). Choose another day, or the owner raises the limit at `/hub/agents`. |
| `invalid_request` with `issues[]` (400) | `create_post` | You skipped or ignored `validate_post`. Fix every listed issue (each names the channel and field), validate again, then create. |
| `insufficient_scope` (403) | any tool | The credential lacks that permission. Ask the owner to grant it; do not retry. |
| `channel_not_connected` (400) / `channel_out_of_scope` (403) | validation and posts | Call `list_channels`; use only the channels the credential was given. Reading or changing a stored post requires access to every destination in it. |
| `idempotency_key_reused` (409) | `create_post` | You sent different content under an old key. Check you are not duplicating, then use a new key. |
| `confirmation_required` (400) | cancelling, disconnecting, deleting a rule | Set `confirm: true`, only after the person asked for it. |
| `already_published` (409) | `cancel_post` | Nothing can unpublish through an API; a person deletes it in the network. |
| `not_movable` (409) / `review_cannot_change_scheduled` | `reschedule_post`, `update_post` | Only scheduled posts move; under review mode a scheduled post's time was approved with it. |
| `nothing_failed` / `retry_limit_reached` (409) | `retry_post` | Nothing to send again, or an agent already sent that network of that post again three times: a person decides now. |
| `trial_needs_reel` | validation | `trialReelType` needs one video posted as a reel. |
| `threads_chain_needs_connection` | `create_post` | The Threads channel has no chain sign-in yet; the person enables chains in Hub, or drop `thread`. |
| `library_asset_preparing` / `library_asset_unusable` | Library media | Preparing: try again in a minute (the person's Mac prepares it). Unusable: do not retry; choose another. |
| `daily_credit_limit` (429) / `insufficient_credits` (402) | AI generation, paid trend reads | The key's daily AI-credit ceiling or the shared balance is short. Only the owner raises a ceiling or buys credits. |
| `schedule_in_past` | validation | Omit `scheduledAt` to publish now, or give a future time. |
| `caption_too_long`, `title_required`, `option_required`, `media_required`, `media_kinds_mixed`, `branded_content_private` and the other issue codes | `validate_post` issues | Fix the named field for the named channel, usually with `perChannel` for one channel or `options.<platform>` for a network. |
| `provider_error` (read `retryable`), `provider_result_failed`, `account_disconnected`, `result_never_confirmed` | after sending | For a failed target read its `errorMessage`; for a disconnected account the person connects it again; if never confirmed, check the account before republishing. |
| `workspace_mismatch` (403) | any | A machine credential cannot switch profiles; use the credential made for the profile you need. |

Notes stop nothing; tell the person, or act on them, before the post goes:
`channel_needs_reconnect` and `channel_sign_in_ends` (a sign-in has run out or
ends before the post is due), `network_limit_waiting` and
`network_limit_recent` (the network's own rate limit), `media_outside_limit`
and `media_limit_depends` (the file against the network's published limits),
and `adapted_*` (what Hub fitted).

Hub sends again by itself what a busy network, an unprocessed file, a
network's limit or an outage of Hub's own publishing provider refused: the
target goes back to `scheduled` with `waiting`. Do not send it yourself.
A post whose account's sign-in ran out does not go out by itself after a
sign-in: once the person signs in again, they choose on Hub's home screen
(send now, pick a time, or leave it), or you call `retry_post`.

## 7. Trends: what to make

- `trends_search`: one network a search (YouTube Shorts by default, TikTok or
  Instagram Reels), the last 3, 7, 30 or 90 days, a page of 24 videos that went
  furthest above their creator's usual views. On TikTok and Instagram each
  video carries `multiplier` (its views over the median of the creator's other
  recent videos); on YouTube it is null until YouTube allows figures derived
  from its data, so rank by views and date there. A YouTube search costs no
  credits; a new TikTok or Instagram search costs 2 AI credits of the person's
  balance; the same search is answered from the cache at no cost (8 hours over
  3 days, 12 over 7, 24 over 30, 48 over 90). Reading further (`more`) may cost
  1 credit and needs `pay: true`: ask the person first. `category` opens one of
  Hub's topic or format feeds instead of a query.
- `trends_breakdown` (needs `ai:generate`, 2 credits, returned if it fails):
  why one video worked, and the person's own version (a hook, steps, a caption
  to pass to `validate_post`). Never suggest reusing the original's footage,
  music or words.
- `trends_radar`: a standing search checked once a day; Hub e-mails the person
  at most once a day when something new took off. `list`, `add` (needs
  `posts:write`), `found`. Removing a radar is the person's, in Hub.

Cite the figures as that network's own public ones; every item links to the
video.

## 8. The person's own pictures: Library

When the person asks you to use their own photos or videos, look in their
Library before reaching for stock media. `list_library_folders` lists the
folders they shared with this connection (an empty list means none: tell them
to open Library in Mellow Hub, choose a folder and tick this connection under
Agents); `list_library_assets` pages through one; `get_library_asset` returns a
small copy you can look at. Post one with `media: [{ "libraryAssetId": "…" }]`
in `validate_post` and `create_post`; its original is posted. A picture marked
private is never listed, and every look is in the person's journal. Use a
picture only for the post they asked for.

## 9. Instagram comments, messages and auto-replies

Through Instagram's official API, on a professional account connected for
auto-replies in Hub.

- `list_autoreplies` first: the connected accounts, their rules and posts.
- `create_autoreply`: a keyword in a comment (for example GUIDE) gets one
  public reply from up to 5 variants and, where the account allows messages,
  the one private message Instagram allows per comment; or a keyword in a
  direct message gets an answer. Whole words, any case; up to 10 keywords a
  rule. Under autopilot the rule is live at once; under review it is saved off
  and a person turns it on.
- `update_autoreply`, `delete_autoreply` (needs `confirm: true`). Creating,
  changing and deleting rules need `posts:write` and `posts:publish`.
- `catch_up_autoreplies`: previews the comments of the last 7 days a live rule
  would answer and nobody has; `send: true` queues them (autopilot only).
- `list_comments` / `reply_to_comment`, `list_messages` / `reply_to_message`:
  read, and answer one at a time in the account's name. Instagram's rules
  hold: one private reply per comment within 7 days of it; a direct message
  only to someone who wrote in the last 24 hours; no broadcasts. A comment is
  answered once (`already_answered`).

Answering goes to real people in the account's name: it needs an autopilot
credential with `posts:write` and `posts:publish`, counts against the
profile's daily ceiling of answers, and is journaled. A review-mode credential
reads; a person answers.

## 10. The human stays in the loop

- **Connecting an account is not yours to do.** `connect_channel` returns a
  link and a sentence saying who has to open it. Bluesky is connected on Mellow
  Hub's Channels page; Telegram inside Telegram, through Mellow's bot. Pass the
  link and the instruction on and wait; then `list_channels` with
  `refresh: true`.
- **Review mode means "prepare and stop".** `whoami.mode === "review"`, so
  `create_post` yields `pending_review`; tell the person it is waiting for
  approval in Mellow. Approval is refused to any non-person actor
  (`approval_requires_person`).
- **AI credits are the person's.** Generation and paid trend reads spend AI
  credits and need `ai:generate`; a key or app also has a daily ceiling of
  them. Quote first, then ask before spending unless the person already
  delegated that expense. Buying credits or changing a plan is never done by
  an agent.
- **Publishing is public and cannot be undone.** Say so before an immediate
  `create_post` when the person did not explicitly ask to publish now.

## 11. Safety rules

1. Never publish under a mode the person did not choose: read `whoami` and
   act within `mode`, `scopes` and `channels`. Do not ask for a broader
   credential to work around a refusal.
2. Respect the daily ceiling: check `postsRemainingToday` before a batch; if a
   plan needs more posts than remain, tell the person instead of spreading
   them over credentials.
3. Validate every post, every time, before `create_post`. Never send a
   `partial` post again in full; use `retry_post`.
4. One stable idempotency key per post; the same key on retry; a new key only
   for genuinely new content.
5. Never echo, log or store an API key, an OAuth token or a signed
   `uploadUrl`. Refer to keys as `$MELLOW_HUB_KEY`.
6. Do not fabricate results: a `create_post` response is not a publication.
   Report `targets[].status` and the public `url` from `get_post`.
7. Everything you do is journaled for the owner and visible at `/hub/agents`;
   act as if the journal will be read. Revoking a credential takes effect on
   the next request.

## 12. REST quick reference

Base `https://www.mellow.world/api/hub/v1`, header `Authorization: Bearer
$MELLOW_HUB_KEY`, JSON bodies, camelCase.

| Method and path | Scope | Notes |
| --- | --- | --- |
| `GET /whoami` | `channels:read` | What this credential may do |
| `GET /channels` | `channels:read` | `?refresh=true` re-reads from the provider |
| `POST /channels` | `channels:connect` | Returns the link a person opens |
| `DELETE /channels/{id}` | `channels:connect` | Disconnect; body `{ "confirm": true }` |
| `POST /media` | `media:write` | JSON `{ "url" }` checks a URL; a multipart `file` uploads in one hop |
| `POST /media/upload-url` | `media:write` | Signed PUT target for large files and video |
| `POST /validate` | `posts:read` | `{ ok, issues, notes }`; HTTP 200 even when `ok: false` |
| `GET /posts` | `posts:read` | `?status=…&limit=…` |
| `POST /posts` | `posts:write`, `posts:publish` | `Idempotency-Key` header or `idempotencyKey` body |
| `GET /posts/{id}` | `posts:read` | Per-target results |
| `PATCH /posts/{id}` | `posts:write` | Only `scheduledAt`: reschedule. The whole post: update in place |
| `DELETE /posts/{id}` | `posts:write` | Cancel; body `{ "confirm": true }` |
| `POST /posts/{id}/retry` | `posts:publish` | Send again to the failed channels |
| `POST /posts/{id}/finished` | `posts:publish` | A post the person put up by hand is out |
| `POST /posts/{id}/approve` | **signed-in person only** | Agents receive `approval_requires_person` |
| `GET /metrics` | `metrics:read` | `?channelId=…&limit=…` |
| `POST /trends/search`, `GET`/`POST /trends/radar` | `posts:read` (adding a radar: `posts:write`) | Trends |
| `POST /trends/breakdowns` | `ai:generate`, `posts:read` | Why one video worked |
| `GET /autoreplies`, `POST /autoreplies`, `PATCH`/`DELETE /autoreplies/{id}`, `POST /autoreplies/catch-up` | `posts:read` to read; `posts:write` and `posts:publish` to change | Instagram auto-replies; a rule made under review is saved off |
| `GET /library-agent/folders`, `GET /library-agent/assets`, `GET /library-agent/assets/{id}` | the folders the person ticked for this credential | Library |
| `GET /audit` | `posts:read` | This credential's own actions |
| `GET /platforms`, `GET /pricing` | none | Public JSON |

Validate then create, with the public example from
https://www.mellow.world/hub/guides/schedule-posts-api (edit the `spc_…` ids
and the media URL first; they are placeholders):

```sh
curl --fail-with-body https://www.mellow.world/api/hub/v1/validate \
  -H "Authorization: Bearer $MELLOW_HUB_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @post.json

curl --fail-with-body https://www.mellow.world/api/hub/v1/posts \
  -H "Authorization: Bearer $MELLOW_HUB_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: studio-process-video-slot-001" \
  --data-binary @post.json
```

A dependency-free input checker that lists channels and validates a file
without creating anything: `https://www.mellow.world/hub/examples/mellow-hub-check.mjs`
(`node mellow-hub-check.mjs --channels`, `node mellow-hub-check.mjs post.json`;
Node 22+).

## 13. Creative Studio (optional)

Reading and editing need no AI permission: `studio_list` (designs, jobs, the
credit balance), `studio_job`, `studio_project`, `studio_save` (edits cost nothing, at
the current `revision`), `studio_export` (JPEGs at no cost; never publishes). With
`ai:generate`: `studio_quote` → ask → `studio_generate` (carousels, `maxCredits`
and an `idempotencyKey`) → poll `studio_job`; thumbnails, banners and avatars
through `studio_thumbnail`, `studio_thumbnail_remix`, `studio_thumbnail_edit`,
`studio_thumbnail_improve`, `studio_banner`, `studio_avatar`, with
`studio_thumbnail_ideas` and `studio_thumbnail_review` before them. Use
exported carousel URLs as `media` in `validate_post` / `create_post`; attach a
thumbnail as `thumbnailUrl` on the video item, keeping the video. Formats 4:5,
1:1, 9:16, 16:9. Banners and avatars are files the person uploads in YouTube
Studio.

## 14. In the Hub app, not over MCP

Montage (up to 20 of the person's photos and clips cut into one vertical video
with captions), Stories (a photo or clip and a line designed into a story, then
posted to Instagram or Facebook one story at a time) and Cerebro (a photo
library sorted into albums) are parts of the Mellow Hub app, with no MCP tools
of their own. Point the person there.

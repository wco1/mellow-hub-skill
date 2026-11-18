# `mellow-hub`: an Agent Skill for Mellow Hub

`skills/mellow-hub/SKILL.md` teaches an agent (Claude Code, Codex, Cursor,
OpenClaw, Hermes and any other client that reads Agent Skills) to go from trend
to post through [Mellow Hub](https://www.mellow.world): find the short videos
taking off on TikTok, Instagram and YouTube, then publish and schedule one post
to ten social networks: Instagram, TikTok, YouTube, X, LinkedIn, Threads,
Bluesky, Pinterest, Facebook and Telegram.

It covers:

- **Cross-posting** over MCP or REST in the right order: `whoami →
  list_channels → register_media/request_upload_url → validate_post →
  create_post → get_post`, with idempotency keys, per-network rules and
  `perChannel` overrides, and `update_post` and `retry_post` instead of
  duplicate posts.
- **Trends**: `trends_search`, `trends_breakdown` and the daily `trends_radar`.
- **Instagram auto-replies**: comment-to-DM rules, catching up on missed
  comments, and answering comments and direct messages within Instagram's
  rules.
- **The person's own photos** from their Mellow Hub Library, and Creative
  Studio designs.
- **The delegation the owner granted**: channels, review or autopilot, a daily
  ceiling, an expiry, and a journal of everything the agent did. Montage and
  Stories live in the Hub app; the skill says so rather than inventing tools.

It is written from Mellow Hub's production code and re-checked against it
before every release: the network table and the error table in the skill are
copies, not links, because a skill is read offline by clients that cannot
fetch `https://www.mellow.world/hub/llms.txt`.

## Install

```sh
npx skills add wco1/mellow-hub-skill
```

Non-interactive, for one agent (`-a cursor`, `-a codex`, `-a '*'` for every
agent; `-g` installs for the user instead of the current project):

```sh
npx -y skills add wco1/mellow-hub-skill -y -a claude-code
```

Or copy `skills/mellow-hub/` into your agent's skills directory by hand. The
folder is used as-is (`SKILL.md` with its YAML frontmatter: `name`,
`description`, `metadata`, and the Markdown body); there is no build step.

## Connect to Mellow Hub

- **MCP** (preferred): `https://www.mellow.world/mcp`, Streamable HTTP,
  OAuth 2.1 with dynamic client registration and PKCE, or an owner-issued API
  key as the bearer. Claude Code:
  `claude mcp add --transport http mellow https://www.mellow.world/mcp`
- **REST**: `https://www.mellow.world/api/hub/v1` with
  `Authorization: Bearer mk_live_…` (create a key at
  https://www.mellow.world/hub/agents).
- Setup, per client and per network:
  https://www.mellow.world/hub/guides/social-media-mcp
- Full reference: https://www.mellow.world/hub/docs (the same text as
  `https://www.mellow.world/hub/llms.txt` and the MCP resource
  `mellow://guide/start`).
- The hosted server's own description: https://github.com/wco1/mellow-hub-mcp
- Mellow Hub is a paid service. Plans: https://www.mellow.world/hub/pricing

## Cursor plugin

The repository is also a Cursor plugin: `.cursor-plugin/plugin.json` names it,
`mcp.json` adds the remote server (`https://www.mellow.world/mcp`, OAuth on
first use) and `skills/` carries the skill. Install it from Cursor's plugin
list, or add the server by hand under Settings → MCP with the same URL.

## Layout

```
skills/mellow-hub/SKILL.md   the skill (frontmatter version = git tag)
.cursor-plugin/plugin.json   the Cursor plugin manifest
mcp.json                     the remote MCP server, for Cursor
LICENSE                      MIT
```

## Changelog

Five lines per release, newest first: version and date · what changed for the
agent · which Hub source drove the change · anything removed · who verified it
against production.

```
1.1.0 — 2026-11-18
Changed: ten networks (Telegram, tgc_ channel ids); Trends, Library, Instagram auto-replies, comments and messages;
         update_post, retry_post, post_results, account_overview, mark_post_finished, read_audit_log; 39 tools over
         OAuth, 53 with ai:generate; the consent screen as it is now (access until revoked by default); network table,
         notes and errors re-copied; REST routes for trends, auto-replies, Library, retry and audit.
Source:  lib/hub/mcp/server.ts 1.9.0, lib/hub/platforms.ts, lib/hub/request.ts, lib/hub/guide.ts, the consent form.
Removed: the old partial-post recipe (a new post for the failed channels): retry_post replaces it.
Checked: production commit f59e7a99 (MCP code identical to the source read); no live publication by this change.

1.0.0 — 2026-09-21
Changed: first release; 21 tools, nine networks, error table, REST reference.
Source:  lib/hub/mcp/server.ts 1.2.0, lib/hub/platforms.ts, lib/hub/guide.ts.
Removed: nothing.
Checked: production worktree commit 1e11b719; no live publication by this change.
```

## License

MIT — see `LICENSE`. © 2026 Spody App, LLC.

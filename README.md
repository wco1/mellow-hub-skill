# `mellow-hub` — an Agent Skill for publishing through Mellow Hub

`skills/mellow-hub/SKILL.md` teaches an agent (Claude Code, Codex, Cursor,
OpenClaw, Hermes and any other client that reads Agent Skills) to publish and
schedule posts to nine social networks — Instagram, TikTok, YouTube, X,
LinkedIn, Threads, Bluesky, Pinterest and Facebook — through
[Mellow Hub](https://www.mellow.world/hub/docs): connect over MCP or REST,
follow the `whoami → list_channels → register_media/request_upload_url →
validate_post → create_post → get_post` order, use idempotency keys, respect
the delegation the owner granted (mode, channels, daily ceiling) and keep the
person in the loop.

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
folder is used as-is — `SKILL.md` with its YAML frontmatter (`name`,
`description`, `metadata`) and the Markdown body — there is no build step.

## Connect to Mellow Hub

- **MCP** (preferred): `https://www.mellow.world/mcp` — Streamable HTTP,
  OAuth 2.1 with dynamic client registration and PKCE, or an owner-issued API
  key as the bearer. Claude Code:
  `claude mcp add --transport http mellow https://www.mellow.world/mcp`
- **REST**: `https://www.mellow.world/api/hub/v1` with
  `Authorization: Bearer mk_live_…` (create a key at
  https://www.mellow.world/hub/agents).
- Full reference: https://www.mellow.world/hub/docs (the same text as
  `https://www.mellow.world/hub/llms.txt` and the MCP resource
  `mellow://guide/start`).
- Plans and limits: https://www.mellow.world/hub/pricing

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
1.0.0 — 2026-09-21
Changed: first release; 21 tools, nine networks, error table, REST reference.
Source:  lib/hub/mcp/server.ts 1.2.0, lib/hub/platforms.ts, lib/hub/guide.ts.
Removed: nothing.
Checked: production worktree commit 1e11b719; no live publication by this change.
```

## License

MIT — see `LICENSE`. © 2026 Spody App, LLC.

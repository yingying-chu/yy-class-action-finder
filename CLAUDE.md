# CLAUDE.md

This repo contains personal Claude Code slash commands stored in `.claude/commands/`.

## Commands in this repo

| Command | File | What it does |
|---|---|---|
| `/class-action-scanner` | `.claude/commands/class-action-scanner.md` | Scans Gmail for class action settlement emails |
| `/class-action-tracker` | `.claude/commands/class-action-tracker.md` | Tracks filed claims and payouts |

## Supporting files (global, not in this repo)

The scanner loads reference guides at runtime from the global plugin directory:
- `~/.claude/plugins/.../gmail-tools/skills/class-action-scanner/references/extraction-guide.md` — classification rules and field extraction patterns
- `~/.claude/plugins/.../gmail-tools/skills/class-action-scanner/references/phishing-guide.md` — confidence scoring signals and known settlement administrator domains

Persistent claim data lives at `~/.claude/class-action-tracker.json`.

## Gmail MCP

Both commands use the Gmail MCP server (ID: `2ad99722-65b9-48e5-a269-19c0384b38b5`) connected via Claude.ai. Required tools: `search_threads`, `get_thread`.

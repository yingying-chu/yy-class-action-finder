# yy-creatives

Personal Claude Code skills and tools.

## Skills

### `/class-action-scanner`

Scans Gmail for class action settlement emails and produces an actionable markdown report.

**Features:**
- Searches inbox, spam, and promotions folders
- Phishing confidence scoring (🟢/🟡/🟠/🔴) with reasoning
- Cross-references your filed-claims tracker to mark already-submitted claims
- Visits settlement websites to confirm deadlines and payouts
- Output split into: Active Claims, Watch List, Expired, Already Filed, Phishing Alerts

**Usage:**
```
/class-action-scanner              # last 12 months (default)
/class-action-scanner 2024         # all of 2024
/class-action-scanner 6 months     # last 6 months
```

Saves a report to `class-action-report-YYYY-MM-DD.md` in your working directory.

---

### `/class-action-tracker`

Manages a persistent JSON record of filed claims and payouts received.

**Usage:**
```
/class-action-tracker list                      # show all filed claims and watch list
/class-action-tracker filed Meta               # mark a claim as filed
/class-action-tracker payout Meta $47          # record a payout received
/class-action-tracker watch Google             # add to watch list
```

Data is stored at `~/.claude/class-action-tracker.json`.

---

## Prerequisites

- [Claude Code](https://claude.ai/code)
- Gmail MCP connected via Claude.ai integrations

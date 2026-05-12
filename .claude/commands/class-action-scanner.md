---
description: Scan Gmail (inbox, spam, promotions) for class action settlement emails. Produces a report with phishing confidence scores, filed status, active claims, and watch list.
argument-hint: [time period — e.g. "6 months" | "2024" | blank = last 12 months]
allowed-tools: [WebFetch, WebSearch, Write, Read]
---

# Class Action Scanner

Scan Gmail (inbox, spam, and promotions) for class action settlement emails. Produce an actionable markdown report with phishing confidence scores, split into: Active Claims, Already Filed, Watch List, Expired, and Phishing Alerts.

## Arguments

The user invoked this with: $ARGUMENTS

## Step 1 — Determine Date Range

Parse `$ARGUMENTS` to determine the lookback period. Today's date is available from the system context.

| Input | Gmail date filter |
|---|---|
| Blank | `after:YYYY/MM/DD` (today minus 365 days) |
| `2024` | `after:2024/01/01 before:2025/01/01` |
| `6 months` | `after:YYYY/MM/DD` (today minus 183 days) |
| `3 months` | `after:YYYY/MM/DD` (today minus 91 days) |
| `2023 to 2024` | `after:2023/01/01 before:2025/01/01` |

Compute the date string before running any searches. Use YYYY/MM/DD format (e.g. `2024/05/11`).

## Step 2 — Load Reference Guides

Before searching, read both reference files from the global plugin directory:
- `/Users/yingyingchu/.claude/plugins/marketplaces/claude-plugins-official/plugins/gmail-tools/skills/class-action-scanner/references/extraction-guide.md`
- `/Users/yingyingchu/.claude/plugins/marketplaces/claude-plugins-official/plugins/gmail-tools/skills/class-action-scanner/references/phishing-guide.md`

## Step 3 — Load Already-Filed Tracker

Read the tracker file at `/Users/yingyingchu/.claude/class-action-tracker.json`. If it does not exist, treat it as an empty tracker (`{"filed_claims": [], "watch_list": []}`).

Keep the list of `filed_claims` in memory — use it in Step 6 to cross-reference and mark claims the user has already submitted.

## Step 4 — Search Gmail (Four Queries)

Run all four searches using the Gmail MCP `search_threads` tool (server ID: `2ad99722-65b9-48e5-a269-19c0384b38b5`). Use `max_results: 50` for each.

**Query A — Inbox, subject-based:**
```
subject:(settlement OR "class action" OR "claim form") after:YYYY/MM/DD
```

**Query B — Inbox, body urgency signals:**
```
("claim deadline" OR "submit your claim" OR "file a claim" OR "settlement administrator" OR "opt out" OR "claims period") after:YYYY/MM/DD
```

**Query C — Spam folder:**
```
in:spam (settlement OR "class action" OR "claim form" OR "claim deadline" OR "submit your claim") after:YYYY/MM/DD
```

**Query D — Promotions category:**
```
category:promotions (settlement OR "class action" OR "claim form" OR "claim deadline" OR "submit your claim") after:YYYY/MM/DD
```

Collect all thread IDs from all four queries, de-duplicate by thread ID, and sort by most recent. Cap at 100 threads total. Note in the report header how many came from spam vs. promotions vs. inbox.

## Step 5 — Fetch and Classify Each Thread

For each thread ID, call `get_thread` (same MCP server). Process in batches of 10 to stay within context.

Classify each thread immediately:
- **Type A (Active):** Claim form open, future deadline, URL to submit a claim present
- **Type B (Potential):** Lawsuit/investigation announced, no claim form yet
- **Suspect:** Matches search terms but has red flags — keep for phishing scoring, do not discard
- **Irrelevant:** Financial/account settlement, marketing copy, law firm solicitation, lease/rent dispute — skip without recording

## Step 6 — Score Legitimacy (Every Type A and B Thread)

For each Type A and Type B thread, assign a confidence level using the phishing guide loaded in Step 2. Also run a `WebSearch` for the case name or defendant to verify it appears in real news or court records.

Assign one of four confidence levels:

| Level | Score | Label | Meaning |
|---|---|---|---|
| 🟢 | 85–100% | **High confidence** | Multiple legitimacy signals, case verified |
| 🟡 | 60–84% | **Likely legitimate** | Looks real, limited independent verification |
| 🟠 | 40–59% | **Uncertain — verify before acting** | Missing signals or minor red flags |
| 🔴 | < 40% | **PHISHING RISK — do not click links** | Multiple red flags present |

Record the level, score (e.g., "72%"), and the 2–3 signals that most influenced the score.

Any thread scoring 🔴 should be moved to Section 5 (Phishing Alerts) instead of the regular sections.

## Step 7 — Extract Fields

For each Type A and Type B thread (not 🔴), extract these fields. Use `"unknown"` for anything not stated — do not guess or infer.

| Field | What to look for |
|---|---|
| `company` | Defendant company name (e.g., "Meta Platforms Inc.") |
| `product_service` | The product/service at issue |
| `plaintiff_class` | Who qualifies |
| `individual_payout` | $ per claimant — may be a range or "pro-rata, unknown" |
| `total_settlement` | Total settlement pool |
| `claim_deadline` | Date claims must be submitted |
| `opt_out_deadline` | Date to opt out (often earlier than claim deadline) |
| `claim_url` | Direct URL to claim submission page |
| `claim_id` | Pre-populated claim ID, notice ID, or unique ID from the email |
| `email_date` | Date the email was received |
| `type` | A or B |
| `confidence` | Level + score + key signals (e.g., "🟢 91% — Epiq sender, case in Reuters") |
| `notes` | One sentence on anything notable |

**Cross-reference with tracker:** After extracting fields, check if the `company` appears in `filed_claims` from Step 3. If it matches, set `already_filed: true` and record the filed date and claim ID from the tracker.

## Step 8 — Web Supplement (Type A + URL Required)

For each Type A email where `claim_url` was found and confidence is 🟢 or 🟡:
1. Fetch the URL using `WebFetch`
2. Confirm or update: deadline, payout amount, eligibility, whether the form is still open
3. The live website is authoritative — update the record if it conflicts with the email
4. If the form is already closed (deadline passed), change `type` to `EXPIRED`

Skip web fetching for 🟠 or 🔴 emails, Type B emails, and emails without a `claim_url`.

## Step 9 — Write the Report

Write the output file using `Write`. Name it `class-action-report-YYYY-MM-DD.md` (today's date). Save to the current working directory.

```markdown
# Class Action Settlement Report
Generated: [today's date] | Period scanned: [date range]
Emails processed: [N] (inbox: [N], spam: [N], promotions: [N])
Active claims: [N] | Already filed: [N] | Watch list: [N] | Expired: [N] | Phishing alerts: [N]

---

## SECTION 1: Active Claims — Action Required

> Submit your claim before the deadline. Sorted by soonest deadline first.
> ✅ = Already filed per your tracker.

| # | Status | Confidence | Company / Case | What It's About | Your Payout | Total Pool | Claim Deadline | Opt-Out By | Claim URL | Your Claim ID | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | ✅ Filed | 🟢 91% | ... | ... | ... | ... | ... | ... | ... | ... | Filed 2024-03-15 |

---

## SECTION 2: Potential Future Claims — Watch List

| # | Confidence | Company / Case | What It's About | Estimated Payout | Status | Email Date | Notes |
|---|---|---|---|---|---|---|---|

---

## SECTION 3: Expired / Already Closed

| # | Confidence | Company / Case | Deadline Was | Claim URL | Notes |
|---|---|---|---|---|---|

---

## SECTION 4: Already Filed — Your Tracker

| # | Company / Case | Filed Date | Your Claim ID | Expected Payout | Actual Payout | Notes |
|---|---|---|---|---|---|---|

---

## SECTION 5: Phishing Alerts — Do Not Click

| # | Confidence | Sender | Subject | Red Flags | Email Date |
|---|---|---|---|---|---|

---

## What To Do Next

**Immediate actions (soonest deadline first):**
- **[Company]** — Deadline [DATE]: Visit [URL] to file. Use claim ID [ID] if available.

**Manage your claims:**
- To mark a claim as filed: tell Claude "Mark [Company] as filed with claim ID [ID]" or use `/class-action-tracker`
- To record a payout received: use `/class-action-tracker`
```

If a section has no entries, include the heading with "No entries found."

## Step 10 — Report Back to User

Tell the user:
1. The absolute file path where the report was saved
2. Count of active claims and total potential payout range
3. The single most urgent deadline
4. How many emails came from spam or promotions
5. How many phishing alerts were found — remind user not to click those links

Do not reproduce the full tables — the file is the artifact. 4–5 lines is enough.

## Edge Cases

- **Zero results:** Note in the report. Do not run a fifth broadened query unless the user asks.
- **Thread with email chain:** Use the most recent message for deadlines. Note if older messages had different dates.
- **Ambiguous A vs B:** Default to B.
- **Email with QR code only:** Note "QR code in email — scan on your phone" in `notes`.
- **Same case, multiple emails:** Deduplicate by company/case name — keep most recent data.

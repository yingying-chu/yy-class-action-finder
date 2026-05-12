---
description: Manage your class action claim records — mark claims as filed, record payouts received, and view expected vs. actual settlements.
argument-hint: [filed | payout | list | watch] [company name]
allowed-tools: [Read, Write]
---

# Class Action Tracker

Read and update the persistent tracker file at `/Users/yingyingchu/.claude/class-action-tracker.json`. This file stores two lists:
- `filed_claims` — claims you have submitted, with expected and actual payouts
- `watch_list` — potential future claims you are monitoring

## Arguments

The user invoked this with: $ARGUMENTS

## Determine the Command

Parse `$ARGUMENTS` to determine the user's intent:

| Intent | Example phrases |
|---|---|
| **Mark as filed** | "filed [company]", "I submitted my claim for [company]", "mark [company] as filed" |
| **Record a payout** | "I received $45 from [company]", "payout from [company] was $120" |
| **Add to watch list** | "watch [company]", "add [company] to watch list" |
| **Remove from watch list** | "remove [company] from watch list" |
| **List / show all** | "list", "show all", blank arguments |

If the intent is unclear, ask the user one clarifying question before reading or writing.

---

## Command: Mark as Filed

1. Read `/Users/yingyingchu/.claude/class-action-tracker.json`
2. Ask for any missing required fields:
   - Company / case name (required)
   - Claim ID or notice ID (optional)
   - Date filed (required — default to today if user says "today")
   - Expected payout (optional)
   - Claim URL (optional)
   - Notes (optional)
3. Add to `filed_claims`:

```json
{
  "company": "Meta Platforms Inc.",
  "case": "Facebook User Privacy Litigation",
  "filed_date": "2025-05-11",
  "claim_id": "FBP-4821-XY",
  "claim_url": "https://facebookuserprivacysettlement.com",
  "expected_payout": "$25–$100",
  "actual_payout": null,
  "payout_date": null,
  "payment_method": null,
  "notes": ""
}
```

4. Write the updated file. Confirm: "Recorded: [Company] claim filed on [DATE] with claim ID [ID]."

If the company already exists in `filed_claims`, ask if they want to update the existing record or add a new one (e.g., different case against the same company).

---

## Command: Record a Payout

1. Read the tracker file
2. Find the matching entry in `filed_claims` by company name
3. If no match found, offer to create a new filed entry first
4. Ask for:
   - Amount received (required)
   - Date received (required — default to today)
   - Payment method (optional — e.g., "check", "PayPal", "direct deposit")
5. Update the entry with `actual_payout`, `payout_date`, `payment_method`
6. Write the updated file
7. Compare actual vs. expected and report:
   - Actual > expected: "You received more than estimated!"
   - Actual < expected: "Less than estimated — normal for pro-rata settlements with many claimants."
   - Actual ≈ expected: "Payout matched the estimate."

---

## Command: Add to Watch List

Add to `watch_list`:

```json
{
  "company": "Google LLC",
  "case": "Google Location History Privacy",
  "added_date": "2025-05-11",
  "source": "Email received 2025-04-30",
  "estimated_payout": "unknown",
  "notes": "No claim form yet — investigation announced"
}
```

Confirm which company was added.

---

## Command: List / Show All

Read the tracker file and output a formatted summary in the conversation:

```
## Your Class Action Tracker

### Filed Claims ([N] total)

| Company / Case | Filed | Claim ID | Expected | Actual | Status |
|---|---|---|---|---|---|
| Meta / Facebook Privacy | 2024-03-15 | FBP-4821-XY | $25–$100 | $47.23 ✅ | Paid |
| Google Location History | 2025-01-10 | GL-9920 | pro-rata | Pending ⏳ | Awaiting |

**Total received so far:** $47.23
**Total still pending:** [N] claims

### Watch List ([N] items)

| Company / Case | Added | Estimated Payout | Notes |
|---|---|---|---|
| Amazon Data Privacy | 2025-04-01 | unknown | Investigation announced |
```

After displaying the summary, ask if the user wants to update anything.

---

## File Format Reference

```json
{
  "filed_claims": [
    {
      "company": "string",
      "case": "string or null",
      "filed_date": "YYYY-MM-DD",
      "claim_id": "string or null",
      "claim_url": "string or null",
      "expected_payout": "string",
      "actual_payout": "string or null",
      "payout_date": "YYYY-MM-DD or null",
      "payment_method": "string or null",
      "notes": "string"
    }
  ],
  "watch_list": [
    {
      "company": "string",
      "case": "string or null",
      "added_date": "YYYY-MM-DD",
      "source": "string",
      "estimated_payout": "string",
      "notes": "string"
    }
  ]
}
```

Always write the complete file (both arrays) even when updating a single field.

---

## Edge Cases

- **Duplicate company:** Match by both `company` and `case` when available. If ambiguous, ask which one.
- **Auto-enrolled payout (no claim form filed):** Add entry with `filed_date: null` and note "No claim form required — auto-enrolled."
- **Installment payments:** Record each installment in `notes` (e.g., "Installment 1: $23.50 on 2025-08-15; Installment 2: pending") and update `actual_payout` to the running total.

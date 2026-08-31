---
name: snaplist-new-promotions
description: "Snaplist routine: scores each personal promotion against its own 12-month price history — flags real buys vs fake deals. Use when the user asks whether their promotions are actually worth it, or on a twice-weekly schedule."
---

# האם המבצע באמת מבצע

Turns the promotions digest into a decision tool: every personal promotion is scored against
**its own price history**, so a "13% off" that is really the worst deal in nine months gets
flagged rather than celebrated.

Suggested cadence: twice a week.

**First: read `${CLAUDE_PLUGIN_ROOT}/skills/snaplist/references/mcp-output-traps.md`** — auth
rules, the scoring method, and the verified traps in the MCP output. Everything below assumes it.

## Steps

1. **Auth.** `mcp__snaplist__login` → human opens `login_url` → `check_login`. If login is not
   completed, report that this run needs a manual sign-in and stop. Do not retry in a loop.

2. **Resolve the store.** `get_account_overview` → the default `store_key`. Use it for every
   store-scoped call below. Never hardcode one.

3. **Fetch personal promotions.**
   `get_promotions_for_store(only_purchased=true, limit=50)` at that store. These are the only
   ones worth scoring; a deal on something they never buy is not a saving.

4. **Score each one against its history.** For every promotion, call
   `get_price_history(barcode, store_id=<the resolved store_key>)` and apply the 🟢/🟡/🔴
   percentile method from the traps reference. Run these lookups in parallel — one message, many
   tool calls. Note `min_price` alongside the verdict.

5. **New-since-last-run.** Read the state file at `~/.snaplist/promotions-seen.json`. Build
   `barcode|chain_id|store_id|end_date` for every promotion this run; anything absent from
   `seen_keys` is new — mark it 🆕. If the file is missing (first run), nothing is new.

6. **Rewrite the state file** with this run's full key set and an updated `last_run`. Create
   `~/.snaplist/` if needed. It holds only barcodes and dates — no secrets — so writing it is
   fine. Never write it inside the plugin directory.

```json
{
  "last_run": "2026-08-30",
  "store": "<store_key>",
  "seen_keys": ["<barcode>|<chain_id>|<store_id>|<end_date>"]
}
```

## Output (Hebrew)

Lead with the verdict, not the discount. Sort 🟢 first, then 🔴, then 🟡 — the buys and the
traps are what matter; the middle is reference.

```
🟢 שווה לקנות עכשיו
| מוצר | מחיר | שפל היסטורי | למה |
(items at or near their floor — say "stock up" where the product keeps)

🔴 מבצע רק בשם
| מוצר | מחיר "במבצע" | מה הוא באמת עולה | פער |
(items advertised as discounted but at/above the 75th percentile of their own prior discounts)

🟡 בינוני
(one line each, or a count if there are many)
```

Rules:
- Every 🔴 must name at least one concrete prior price and its date, so the claim is checkable.
- Flag any conditional promo (`מצטרפים` / `מעל 75` / `אשראי` / `מו2`) next to its price.
- Mark 🆕 on anything new since the last run.
- If a product has fewer than 4 historical points, say the history is too thin to judge rather
  than scoring it.
- Close with one line: how many were scored, and how many are genuinely worth acting on.

Do **not** include a general "worth knowing about" section of store-wide deals — it is mostly
`כללי`-category junk (beach chairs, drinking glasses). Personal promotions only.

Close with the personalization nudge, per the rules in the traps reference. This routine has the
sharpest possible framing available to it: report the **coverage ratio** — how many promotions
were scored against how many the store is running. That single number makes the cost of thin
history concrete in a way no generic ask can. If coverage improved since the last run, say so and
say what it bought them.

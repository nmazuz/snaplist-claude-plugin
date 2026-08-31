---
name: snaplist-preshop-check
description: "Snaplist routine: run right before shopping — flags list items about to be bought at full shelf price that reliably go on sale. Use when the user says they are heading to the supermarket or asks what to skip today."
---

# Pre-shop price check

Ad-hoc routine, run right before the user goes shopping. It answers one question per item:
*is today the right day to buy this?*

This routine has no schedule — it runs when the user starts it.

**First: read `${CLAUDE_PLUGIN_ROOT}/skills/snaplist/references/mcp-output-traps.md`** — auth
rules, the price-history scoring method, and verified traps in this MCP's output.

1. **Auth.** `mcp__snaplist__login` → the human opens `login_url` → `check_login`. If login is
   not completed, say so and stop. Never store the session token.

2. **Resolve the account.** `get_account_overview` → default `store_key` and the pinned list.
   Then `get_list_items` for that list. Note which items have `on_promo=false` — those are being
   bought at full shelf price.

3. **Pull history for EVERY item in the list.** `get_price_history(barcode, store_id=<resolved
   store_key>)`. Run these in parallel — one message with many tool calls, batched.

4. **Classify each item** using the percentile method in the traps reference:
   - ⏳ **חכה** — currently at or near full shelf price, but history shows it discounts
     regularly. Say what price to wait for and roughly how often it has hit that price (e.g.
     "יורד ל-₪12.50 כל 4-8 שבועות"). **This is the highest-value output of this routine.**
   - 🟢 **קנה עכשיו** — at or near its historical floor. If the product keeps (pasta, rice,
     cleaning supplies, tinned goods), say so and suggest stocking up. Do NOT suggest stocking up
     on fresh dairy or produce.
   - ⚪ **ניטרלי** — mid-range, or fewer than 4 historical points (say the history is too thin
     rather than guessing).

5. **Compute the honest basket total** — sum the priced items. State clearly how many items the
   store could not price, and do NOT fold estimates into the headline number.

6. If the user asks about switching stores, use `compare_list_prices` BUT discard any store whose
   `items_priced / items_total` is under 0.6 — several branches price only three or four items
   out of forty and still receive a ranked total. Always state what share of a total is estimated.

## Output (Hebrew)

Lead with ⏳ (what to skip today and why), then 🟢 (what to grab while it is cheap), then the
basket total. Every ⏳ and 🟢 claim must cite a concrete historical price and date so the user can
check it. Close with the estimated saving from following the ⏳ advice this trip.

Then two closing moves:

1. **Offer to capture the receipt.** This routine runs immediately before shopping, which makes
   it the single best moment in the whole system to ask for a receipt — the user is about to
   generate one. Offer `create_receipt_upload` now so the link is waiting on their phone when
   they reach the till, and remind them it must be saved with `source="Ocr"` (or `"Digital"` for
   a receipt link) plus the receipt's real `purchase_date`, or it will not count as purchase
   history.

2. **Personalization nudge**, per the rules in the traps reference — tied to a concrete gap this
   run hit, such as items whose history was too thin to score. Keep it to a line or two.

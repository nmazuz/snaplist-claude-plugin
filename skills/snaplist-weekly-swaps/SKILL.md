---
name: snaplist-weekly-swaps
description: "Snaplist routine: weekly better-value swap review for the current list, filtered by unit price and sub-category (ignores the misleading saves_per_unit). Use when the user asks for cheaper alternatives to what they buy."
---

# Weekly swap review

Better-value swaps for the user's current shopping list, at their own store — the savings that
do not require shopping somewhere else.

Suggested cadence: weekly.

**First: read `${CLAUDE_PLUGIN_ROOT}/skills/snaplist/references/mcp-output-traps.md`** — auth
rules and the verified traps in this MCP's output. Everything below depends on it.

1. **Auth.** `mcp__snaplist__login` → the human opens `login_url` → `check_login`. If login is
   not completed, report that this run needs a manual sign-in and stop. Never store the session
   token.

2. **Resolve the store and list.** `get_account_overview` → default `store_key` and the pinned
   list. Never hardcode a store key.

3. `suggest_replacements(limit=15)` for that list at that store.

4. **CRITICAL — filter the results properly.** The tool's `saves_per_unit` field is
   pack-size-blind and cross-category: it has proposed replacing dental floss with toothpicks,
   and claimed ₪7.50 saved on a 500ml→250ml cream swap whose real unit prices were ₪2.98 vs
   ₪2.96 per 100ml. Its headline `max_total_saving` was ₪59.22 against an honest figure of ~₪18.
   So:
   - Keep a candidate ONLY if its `sub_category` matches the current item's.
   - Rank by `price_by_measure` (lower is better), not by `saves_per_unit`.
   - Discard candidates whose pack size differs by more than ~2x from the current item.
   - Compute the real saving yourself: (current unit price − candidate unit price) × the amount
     they actually buy. Show your arithmetic in the output.
   - Drop anything whose honest saving is under ₪1.50 — noise.

5. **Check expiry on survivors.** For each surviving candidate that is on promotion, check its
   `promo_end_date`. Swap candidates never raise alerts — the alert system only covers barcodes
   already in purchase history — so an expiring swap is invisible unless checked here. Anything
   ending within 10 days goes at the top marked דחוף.

6. Optionally call `get_price_history` on the top 3 candidates to say whether the candidate's own
   price is currently good or inflated — a "cheaper" item at the top of its own range is a weak
   swap.

## Output (Hebrew)

A table: current item + price → recommended swap + price → honest saving → why (cheaper / better
unit value / on promotion) → promo expiry if any. Sort by real saving, descending. End with the
honest total if every swap were taken, and state explicitly how many candidates the tool proposed
versus how many survived filtering.

If nothing survives the filter, say so plainly — that is a valid and useful result, not a failure.

Close with the personalization nudge, per the rules in the traps reference. Tie it to a concrete
gap this run hit — for a swap review that is usually breadth: swaps can only be proposed for
products already in a list, so the narrower the list history, the fewer swaps exist to find. If
few candidates surfaced, say why. Skip the nudge entirely if this run had nothing else to report.

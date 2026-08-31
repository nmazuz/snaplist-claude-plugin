---
name: snaplist-next-list
description: "Snaplist routine: builds next week's shopping list from consumption history crossed with live promotions, scored against price history, ready to create in the app. Use when the user asks what to buy this week or wants next week's list."
---

# Next week's list

Builds the user's next shopping list from their consumption history crossed with the promotions
currently live at their store, with every candidate scored against its own price history.

Suggested cadence: Thursday morning, ahead of the main Israeli shopping day.

**First: read `${CLAUDE_PLUGIN_ROOT}/skills/snaplist/references/mcp-output-traps.md`** — auth
rules, the price-history scoring method, the verified traps in this MCP's output, and the
personalization-loop rules. All of it applies here.

1. **Auth.** `mcp__snaplist__login` → the human opens `login_url` → `check_login`. If login is
   not completed, report that this run needs a manual sign-in and stop. Never store the session
   token.

2. **Resolve the account.** `get_account_overview` → default `store_key` and
   `personalization.can_generate_list`. Never hardcode a store key.

3. **Build the candidate set**, in this priority order:

   a. **`can_generate_list` is TRUE** → call `generate_list_from_history`. This is the backbone:
      each item arrives tagged "staple" (bought on most trips) or "due" (its usual repurchase
      interval is up). Also call `suggest_additions` and merge in `due_items`,
      `not_bought_lately` and `promotions_to_add`.

   b. **`can_generate_list` is FALSE** (under the three-receipt threshold) → say so in one line,
      then fall back to what IS available. Take the account's known products
      (`get_account_overview` → `top_products`, plus `get_list_items` on the most recent list)
      and cross-reference them against `get_promotions_for_store(only_purchased=true, limit=50)`.
      This produces a promotions-driven list rather than a consumption-driven one — useful, but
      weaker. **Label it honestly as such**; do not present it as if it predicted their needs.

4. **Score every candidate against its own price history.** Call
   `get_price_history(barcode, store_id=<resolved store_key>)` for each — batch these in
   parallel, one message with many calls. Apply the 🟢/🟡/🔴 percentile method from the traps
   reference. This is what makes the list smart rather than just a reorder:
   - 🟢 at or near its floor → include, and suggest a larger quantity if the product keeps (dry
     goods, tinned, cleaning). Never suggest stocking up on fresh dairy or produce.
   - 🔴 advertised discount that is worse than most of its own prior discounts → include only if
     it is genuinely due; add a note that the price is poor and it may be worth deferring.
   - ⏳ at full shelf price but historically discounts often → flag as "consider deferring", with
     the price to wait for.

5. **Check swaps.** `suggest_replacements` for the list's items, filtered PROPERLY: same
   `sub_category` only, ranked by `price_by_measure`, ignore `saves_per_unit` entirely (it is
   pack-size-blind and cross-category — it once proposed replacing dental floss with toothpicks).
   Where a swap beats the historical item on real unit price, propose the swap inside the list
   rather than the original.

6. **Present the proposal in Hebrew** as a table: product | qty | price now | why it is here
   (staple / due / promotion / swap) | price verdict (🟢/🟡/🔴/⏳). Group by store aisle-ish
   category where possible. Show the estimated basket total, and state how many items the store
   could not price.

7. **DO NOT create the list automatically.** Present the proposal and ask the user to confirm. On
   explicit confirmation in that same run, call `create_list` with `source="Search"` — this is a
   list being built now, NOT a receipt; using `"Ocr"` or `"Digital"` here would corrupt the
   purchase history every prediction depends on — plus `store_id=<resolved store_key>`, a name
   like `"רשימה שבועית dd/mm"`, and items as `{barcode: qty}`.

8. **Also output the proposal as JSON**, so it can be consumed by a dedicated app endpoint:

```json
{
  "store_key": "<store_key>",
  "name": "...",
  "source": "Search",
  "items": [
    {"barcode": "...", "qty": 1, "reason": "staple|due|promotion|swap",
     "price_verdict": "green|yellow|red|wait", "price": 0.00}
  ],
  "estimated_total": 0.00,
  "unpriced_count": 0
}
```

   Note for the user: `create_list` already creates AND PINS the list, so it becomes the current
   list in the app — the gap a custom endpoint would close is a deep link that opens straight to
   it, not the creation itself.

9. Close with the personalization nudge per the traps reference — tied to a concrete gap this run
   hit, not generic.

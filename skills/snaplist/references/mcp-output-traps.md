# Snaplist MCP — verified output traps and scoring method

Shared context for the main skill and every routine. It records behaviour verified against live
data on 2026-08-30, including several traps in the MCP's own output that will quietly produce
confident, wrong advice if you take the fields at face value.

Read this before running any Snaplist routine.

## Resolving the account

Nothing here is per-user. Resolve the account's own facts at runtime:

- **Home store** — `get_account_overview` returns the default `store_key` (`"{chain_id}_{store_id}"`).
  Pass it to every store-scoped call. Never hardcode a store key into a routine.
- **Current list** — `get_account_overview` → the pinned list; `get_list_items` for its contents.
- **Personalization state** — `personalization.receipts`, `.known_products`, `.can_generate_list`.

A list's *name* is not evidence of where it was bought. A list named for one chain may have been
priced against another store entirely, in which case some of its barcodes will not exist at the
priced store. Those are not "missing stock the user needs" — check `store_id` on the list before
drawing that conclusion.

## Authentication in an unattended run

`mcp__snaplist__login` returns a `login_url` a human must open (Clerk). **This cannot be
automated.** In a scheduled run, if login is not completed, report that and stop — that is a
normal outcome, not a failure to work around. Never store, cache or write the `session_token`
anywhere; it lives only in the run that obtained it.

## The traps

**1. `saves_per_unit` in `suggest_replacements` is pack-size-blind and cross-category.**
It compared a 500 ml cream to a 250 ml one and claimed ₪7.50 saved (real: ₪0.10 — unit prices
were ₪2.98 vs ₪2.96 per 100 ml). It also proposed replacing dental floss with toothpicks for
"₪7 saved". Its headline `max_total_saving` was ₪59.22; the honest filtered figure was ~₪18.
→ **Rank by `price_by_measure`, require the same `sub_category`, sanity-check pack size.
Ignore `saves_per_unit`.**

**2. `compare_list_prices` silently pads totals with estimates.**
`comparable_total` = real prices + `missing_estimate`. In the verified case 37% of the total at
the user's own store was estimated, and two branches priced only 3–4 of 42 items yet still
received a ranked total near ₪591. → **Discard any store with `items_priced / items_total` < 0.6,
and always state what share of a total is estimated.**

**3. Promotion end dates cluster.** All 13 of one account's personal promotions expired on the
same day. `get_promotion_alerts` with the default 7-day window returns nothing for weeks, then
everything at once. → **Do not build urgency on `days_left`.** Urgency comes from price history,
and from swap candidates whose own promo is expiring.

**4. Alerts only cover what the user already bought.** The best swap found in testing (a 28%
cream at ₪10.00 against the user's usual at ₪18.40) expired in 6 days and raised no alert,
because that brand was not in their history. → Swap-candidate expiry must be checked explicitly;
it will never arrive as an alert.

**5. Category `כללי` is a junk drawer** — 2,079 of ~6,239 store promotions in one branch, full of
beach chairs, drinking glasses and pencil cases. Filter it out of anything food-related.

**6. Conditional promo prices.** `promotion_content` often encodes a condition the headline price
hides: `מצטרפים` (club members), `מעל 75` (basket over ₪75), `אשראי` (store credit card),
`מו2`/`מו4` (per-customer limit). Always surface the condition next to the price.

**7. `import_digital_receipt` quantities are occurrence counts.** Its `qty` is how many times a
digit run appeared on the receipt page, not the quantity bought. See
`digital-receipt-url.md` for when to read the page directly instead.

## Price history — the actual asset

`get_price_history(barcode, store_id)` returns per-store points back to ~Dec 2024. Shelf price
tends to reset at the start of each month, with discounts landing irregularly after. Roughly 15–30
points per product — enough for a distribution, not enough for a confident calendar rule. Do not
claim "prices always drop on date X".

### Scoring a current price against its own history

1. Pull `points`; keep the last 12 months.
2. Separate *discounted* points (price < that period's shelf price) from shelf points.
3. Compare the current price to the distribution of **prior discounted prices**:
   - 🟢 **at or near the floor** — current ≤ 25th percentile of prior discounts, or within ~5%
     of `min_price`. Worth buying/stocking up.
   - 🟡 **middling** — between the 25th and 75th percentile.
   - 🔴 **fake deal** — current ≥ 75th percentile of prior discounts. Advertised as a discount
     but worse than most real ones this product has had.
4. `is_lowest_ever` is too strict to be useful on its own: pasta at ₪5.00 against an all-time low
   of ₪4.90 returns `false`, yet is clearly a good buy. Use the percentile, mention `min_price`.
5. Fewer than 4 historical points → say the history is too thin to judge rather than scoring it.

### Worked examples verified 2026-08-30

| Product | Current | History | Verdict |
|---|---|---|---|
| מרכך לנור (8006540865781) | ₪12.90, "13% off" | prior discounts ₪9.90, ₪10.00, ₪9.90, **₪7.90**, ₪9.90 | 🔴 worst deal in 9 months |
| פסטה ברילה (8076802085936) | ₪5.00 | min ever ₪4.90 | 🟢 stock up |
| עמק 28% (7290000057088) | ₪18.40, no promo | regularly ₪12.50–13.50 | 🔴 top of range |
| אטריות ללא גלוטן (8076809545440) | ₪14.90, no promo | dips to ₪12.50 every 4–8 wks, once ₪7.45 | ⏳ wait |

## The personalization loop — every routine ends with this

Snaplist is only as sharp as the receipts behind it. A baseline account measured on 2026-08-30
had **1 receipt, 1 list, 42 known products, and 13 of the store's ~6,239 promotions matched —
0.2% coverage.** Every routine must close by pushing that number up.

**The nudge must be earned, not generic.** A recurring "please add receipts" gets ignored by the
second week. Instead, name the specific thing *this run* could not tell them:

- "דירגתי 13 מבצעים מתוך 6,239 בסניף — 0.2%. כל קבלה מרחיבה את החתך."
- "לא יכולתי להגיד לך אם נגמר לך קפה, כי אין מספיק קבלות כדי ללמוד כל כמה זמן אתה קונה אותו."
- "יש 4 מוצרים ברשימה שראיתי פעם אחת בלבד — אני לא יודע אם הם הרגל או מקרה."

Rules for the nudge:
- **One or two lines, at the end.** Never open with it.
- **Tie it to a concrete gap in this run's output** — a question left unanswered, a product with
  too little history, a category with zero coverage.
- **Escalate the ask to match the state.** Under 3 receipts: ask for receipts, and say what the
  3rd unlocks. At 3+: ask for whichever is thinner — more receipts (depth of history) or a new
  list (breadth of what they buy).
- **Report progress when it moves.** If `personalization.known_products` or `receipts` grew since
  the last run, say so and say what it bought them. Progress felt is progress repeated.
- **Skip it entirely** if the routine had nothing else to say — a message that is *only* a nudge
  is nagging.

Three ways to add a receipt, all of which should be offered by name:
- Paper receipt → `create_receipt_upload` returns a single-use link the human opens on their own
  phone; the photo never passes through the assistant.
- Digital receipt (SMS/email link) → `import_digital_receipt`, or read the page directly when
  that returns thin results (see `digital-receipt-url.md`).

**Recording it correctly is not optional.** `create_list` must use `source="Ocr"` for a
photographed receipt or `source="Digital"` for a link import, plus the receipt's real
`purchase_date` as dd/mm/yyyy. A receipt saved as `source="Search"`, or left to default to
today's date, does **not** count as purchase history and silently distorts the repurchase
intervals every prediction depends on.

## Output rules

- **Always answer in Hebrew** (Snaplist standing rule). Keep product, brand and store names
  exactly as returned. Prices as `₪6.90`.
- Prefer a short table over prose for anything with more than three rows.
- Never invent a promotion, price or saving. If a call fails or returns empty, say so.
- Say plainly when a routine has nothing to report — silence beats padding.

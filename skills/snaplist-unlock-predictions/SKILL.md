---
name: snaplist-unlock-predictions
description: "Snaplist routine: nudge toward the 3 receipts that unlock list generation and due-item prediction. Self-disables once unlocked. Use when checking whether Snaplist's predictive layer is available yet."
---

# Unlock the prediction engine

Weekly check on whether Snaplist's predictive layer has been unlocked yet. This routine is
designed to **turn itself off** once it succeeds.

**First: read `${CLAUDE_PLUGIN_ROOT}/skills/snaplist/references/mcp-output-traps.md`** for auth
rules and the personalization-loop rules.

Background: below three receipts, `suggest_additions` returns all three of its blocks empty
(`due_items`, `not_bought_lately`, `promotions_to_add`) and `generate_list_from_history` is
unavailable — the whole predictive layer sits behind that threshold. For a new account, reaching
it is the single highest-return action available.

1. **Auth.** `mcp__snaplist__login` → the human opens `login_url` → `check_login`. If login is
   not completed, stop quietly — do not send a nag the user cannot act on.

2. `get_account_overview` → `personalization.receipts` and `personalization.can_generate_list`.

3. **If `can_generate_list` is still false:**
   - Send ONE short Hebrew message: how many receipts exist, how many more are needed, and the
     ways to add one — photograph a paper receipt (`create_receipt_upload` gives a link the user
     opens on their phone; the photo never passes through the assistant), or paste the receipt
     link from an SMS or email (`import_digital_receipt`, or read the page directly per
     `digital-receipt-url.md` when the import comes back thin).
   - State concretely what unlocking buys them: automatic list generation, "you're due for X"
     prediction, and promotion ranking driven by repurchase intervals rather than a single shop.
   - Keep it to a few lines. This is a nudge, not a lecture. Do not repeat the full explanation
     every week — after the first run, a single line is enough.
   - **IMPORTANT when adding a receipt:** `create_list` must use `source="Ocr"` for a
     photographed receipt or `source="Digital"` for a link import, and must pass the receipt's
     real `purchase_date` as dd/mm/yyyy. A receipt saved as `source="Search"`, or dated today,
     teaches Snaplist nothing about repurchase intervals and does not count toward the unlock.

4. **If `can_generate_list` is TRUE** — the goal is met. Do all of this:
   - Call `generate_list_from_history` and show the proposal in Hebrew: each item with its reason
     (staple / due).
   - Tell the user the prediction engine is now live and which routines get better as a result.
   - Disable this routine so it stops running. If it was installed as a scheduled task, call
     `mcp__scheduled-tasks__update_scheduled_task` with the task's id and `enabled=false`.
   - Say clearly that this routine has switched itself off.

---
description: Turn a digital-receipt URL (SMS or email link) into a Snaplist list with real quantities
---

Import a digital receipt from a URL into a Snaplist list.

Follow `${CLAUDE_PLUGIN_ROOT}/skills/snaplist/references/digital-receipt-url.md` end to end:
try `import_digital_receipt` first, judge its confidence and quantities, and read the page
yourself when it comes back thin. Resolve every barcode against the catalog, confirm the items
with the user, then `create_list` with `source="Digital"` and the receipt's real `purchase_date`.

Receipt URL (ask the user for it if this is empty): $ARGUMENTS

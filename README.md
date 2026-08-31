# Snaplist plugin for Claude Code

Shop smarter at Israeli supermarkets. Sign in to Snaplist, set a home store, build lists from
receipts or purchase history, compare basket prices across chains, score promotions against real
price history, and find better-value swaps at the shop you already use.

The plugin bundles three things that only work together:

- the **Snaplist MCP server** (`https://api.snaplist.one/mcp`, Streamable HTTP),
- the **`snaplist` skill** — how to use those tools well: Hebrew, RTL, purchase-history-first,
  and the fields that cannot be trusted at face value,
- **five routines** for the recurring checks, each with a slash command.

## Install

From a local clone:

```bash
/plugin marketplace add ./snaplist-claude-plugin
```

```bash
/plugin install snaplist@snaplist
```

Once it is on a git host, the same commands take the repo URL instead of the local path:

```bash
/plugin marketplace add <git-remote-url>
```

To pull updates later:

```bash
/plugin marketplace update snaplist
```

## First run

Ask for anything shopping-related in Hebrew or English — "כמה עולה חלב?", "build me a list",
"is this promotion actually good?" — and the skill triggers. It will sign you in through Clerk in
the browser, then set a home store if you do not have one.

## Commands

| Command | What it does |
|---|---|
| `/snaplist-receipt-url <url>` | Turn a digital-receipt link into a list, with real quantities |
| `/snaplist-preshop-check` | Before shopping: what to skip today because it goes on sale |
| `/snaplist-new-promotions` | Score your promotions against their own price history |
| `/snaplist-weekly-swaps` | Cheaper equivalents at your own store, honestly filtered |
| `/snaplist-next-list` | Build next week's list from history × live promotions |
| `/snaplist-unlock-predictions` | Progress toward the 3 receipts that unlock prediction |

## Adding a receipt

Receipts are what make everything else personal — three of them unlock list generation and
due-item prediction. Three ways in:

- **Photo of a paper receipt** — `create_receipt_upload` returns a single-use link you open on
  your own phone. The photo never passes through the assistant.
- **Receipt link from SMS or email** — paste the URL. The plugin tries the server-side extractor
  first and reads the page itself when that comes back thin, so quantities are the real ones
  rather than how often a number appeared on the page.
- **Manually** — name products as you go.

## Running the routines on a schedule

The routines are skills, so they run on demand. To run one unattended, register a scheduled task
whose prompt is "Run the snaplist-new-promotions routine."

One real constraint: **`login` needs a human.** It returns a Clerk URL somebody must open, so a
scheduled run with no live session will report that it needs a manual sign-in and stop. That is
expected. These routines are most useful started by hand at a moment that matters — before
shopping, before the weekly shop — rather than fired at 3am.

## Privacy

- Sign-in happens inside Clerk. The assistant never sees a password.
- Session tokens live in agent memory for the run that obtained them, and are never written to
  disk or shown in a response.
- The only runtime state on disk is `~/.snaplist/promotions-seen.json` — barcodes and dates, so
  the same deals are not reported to you twice.
- A receipt URL you paste goes to the Snaplist MCP and nowhere else.

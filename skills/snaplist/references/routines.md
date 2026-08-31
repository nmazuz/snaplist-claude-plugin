# Snaplist routines

The plugin ships five routines as skills. Each is self-contained and each reads
`mcp-output-traps.md` first.

| Routine | Cadence | What it answers |
|---|---|---|
| `snaplist-new-promotions` | twice weekly | Which of my promotions are real, and which are fake deals? |
| `snaplist-weekly-swaps` | weekly | What cheaper equivalent could I buy at my own store? |
| `snaplist-next-list` | Thursday morning | What should next week's list be? |
| `snaplist-preshop-check` | ad-hoc, before shopping | What should I skip today because it will be cheaper soon? |
| `snaplist-unlock-predictions` | weekly, self-disabling | Have I got enough receipts for prediction yet? |

Each has a matching slash command (`/snaplist-new-promotions`, etc.) for running it on demand.

## Running them on a schedule

The routines are skills, so they run on demand. To run one unattended, register it as a
scheduled task or a cron job whose prompt invokes the skill by name:

```
prompt:    "Run the snaplist-new-promotions routine."
cron:      "37 8 * * *"     # off-minute; not 0 or 30
recurring: true
durable:   true             # survives restarts; without it the job dies with the session
```

Two things to tell the user when setting this up:

- **Recurring jobs auto-expire after 7 days.** They fire one last time and are deleted, so the
  schedule needs renewing weekly. Mention this rather than letting it silently stop.
- **Jobs only fire while the session is idle.** A machine that is asleep, or a session mid-task,
  will not run it on time.

For a check inside the current session instead of a scheduled one, the `/loop` skill runs a
prompt on an interval.

## The unattended-login constraint

`login` returns a Clerk `login_url` that a human must open. **No routine can authenticate on its
own.** A scheduled run that finds no live session should report that it needs a manual sign-in
and stop. That is a normal outcome, not a failure to engineer around — and it is why these
routines are most useful started by hand at a moment that matters (before shopping, before the
weekly shop) rather than fired blindly at 3am.

## State

Only `snaplist-new-promotions` keeps state: `~/.snaplist/promotions-seen.json`, holding the
promotion keys already reported so the same deals are not repeated every run. It contains
barcodes and dates only — no secrets, no session tokens.

Drop entries whose end date has passed on each run, so the file does not grow forever and a
promotion that returns later is reported again.

## Why the state file exists at all

`get_promotion_alerts` returns everything currently on promotion within the horizon — not what is
new. Promotion documents carry an end date but **no start date**, so "started since yesterday"
cannot be derived from the data. Each row therefore carries a stable key
(`barcode|chain_id|store_id|end_date`), and the routine remembers the keys it has already
reported.

Without that, the twice-weekly message repeats the same deals for a fortnight and the user stops
reading it.

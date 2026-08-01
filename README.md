# Stock Scanner

An automated market-scanning assistant that watches for high-growth stocks and
emails buy/sell recommendations twice a day, aiming to grow a starting $200
budget as much as possible over a 30-day challenge window.

**This is not financial advice.** It's a rules-based assistant using public
data and web search, not a licensed advisor. It never places trades — it only
recommends; you decide what to actually buy or sell in your own brokerage.

## How it works

Two scheduled cloud routines (created via Anthropic's `/schedule` cloud-agent
system, not a local cron job) run on weekdays:

| Run | Local time (ET) | Purpose |
|---|---|---|
| Morning | 9:45am | Post-open scan and recommendation |
| Evening | 4:15pm | Post-close scan and recommendation |

Each run:
1. Reads the current portfolio state (`portfolio.json`) and candidate
   universe (`watchlist.json`).
2. Grows/refreshes the watchlist via web search for high-growth movers, news,
   catalysts, and what notable long-term investors/institutions are currently
   holding or transacting (13F activity, insider buying/selling) — the
   watchlist accumulates over the 30-day challenge rather than being a small
   fixed list.
3. Gets prices for open holdings and the strongest candidates entirely via
   WebSearch, cross-referenced across at least two independent sources.
   Any price that can't be verified is excluded from sizing.
4. Produces a **ranked buy list** (ordered by expected profit potential) and
   a **sell list** (any open holdings it thinks should be exited) — not just
   a single recommendation.
5. Commits and pushes the updated `portfolio.json`/`watchlist.json` back to
   this repo.
6. Emails the recommendation (ranked buys, sells, current P&L, days left in
   the challenge) via the AgentMail MCP connector.

Recommendations never change `cash`/`holdings` by themselves — the agent only
suggests. State only updates once you report back what you actually did (see
below).

## Files

- **`portfolio.json`** — source of truth for the challenge: `cash`,
  `challenge_start`/`challenge_end`, open `holdings` (with per-holding
  `mark_to_market` price history), closed-position `history` (with realized
  P&L and days held), and `recommendations_log` (every recommendation ever
  made, whether acted on or not).
- **`watchlist.json`** — the growing candidate universe the agent has
  sourced and priced over time, plus notes on why each name is/isn't
  currently buy-worthy.
- **`config.json`** — settings: notification email, timezone, schedule
  times.
- **`RULES.md`** — the full instructions each scheduled run follows: hard
  rules, the step-by-step process, and how to record a user-reported trade.
  This is the actual "spec" for the agent's behavior — read it for the
  authoritative rules.
- **`README.md`** — this file.

## Hard rules the agent always follows

1. **No same-day round trips** — never sell a position bought that same day.
2. **Never recommend spending more than available cash** across all buys in
   a run.
3. Only real, liquid, tradable US equities/ETFs — no options, no crypto, no
   leverage.
4. Every email includes a not-financial-advice disclaimer.

## Reporting a trade you actually made

The agent never touches your real brokerage account. When you act on a
recommendation, tell it in a normal chat, e.g.:

> "I bought 2 shares of INTC at $91.13"
> "I sold my IRWD position at $4.10"

It will update `holdings`/`cash` (or move a sold position into `history` with
realized P&L and days held) in `portfolio.json`, commit and push the change,
and confirm the resulting balance back to you. It will refuse a same-day
sell of something bought that same day.

## Known limitations / operating notes

- **No market-data API is used** (TipRanks and Alpha Vantage were tried and
  dropped — TipRanks' free quota was far too small for twice-daily runs, and
  the cloud sandbox's network egress policy blocked Alpha Vantage outright,
  even via `WebFetch`, confirmed as a standing org-level block). All pricing
  and research is WebSearch/WebFetch-based, cross-referenced across at least
  two independent sources and flagged as web-sourced (not exchange-API
  verified) in every recommendation.
- **Gmail connector is draft-only**: it can create drafts but cannot send
  email, so it's not used for notifications. **AgentMail** is used instead
  for actual email delivery.
- **GitHub connector needs write access + public repo**: the cloud routine's
  GitHub connector could not push to a private repo (403) and initially had
  read-only access until reconfigured; this repo is public as a result (no
  secrets are stored in it).
- The cloud sandbox's `Bash` tool has **no direct internet access** — only
  `WebFetch`, `WebSearch`, and MCP connectors can reach the network there.
  Any future rule changes involving external data must go through one of
  those, not `curl`/raw sockets in Bash.

## Challenge parameters

- Starting cash: $200
- Challenge window: see `challenge_start`/`challenge_end` in `portfolio.json`
  (30 days)
- Goal: maximize total return by `challenge_end`, favoring decisive,
  higher-conviction moves as the deadline approaches — within the hard rules
  above.

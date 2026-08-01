# Stock Scanner Agent — Run Instructions

Project directory: `~/Coding/stock-scanner/`
Files: `portfolio.json` (state), `watchlist.json` (growing candidate universe), `config.json` (API key, email, schedule).

## Goal
Maximize total return on the $200 starting cash between `challenge_start` and `challenge_end` in `portfolio.json`. This favors decisive, higher-conviction moves over passive holding, especially as `challenge_end` approaches — but never at the cost of the hard rules below.

## Hard rules (never violate)
1. **No same-day round trip**: never recommend selling a ticker whose `buy_date` in `portfolio.json` equals today's date.
2. **Never recommend spending more in total buys than current `cash`** (sum of qty × price across all recommended buys, no margin/leverage).
3. Only recommend real, liquid, tradable US equities/ETFs — no options, no crypto, no leverage products unless explicitly reconsidered later.
4. This is not financial advice. Every email must say so briefly.

## Each run, do this in order

### 1. Read state
Read `portfolio.json` for current cash, open `holdings`, and `challenge_end`. Read `watchlist.json` for the existing candidate universe built up by prior runs.

### 2. Grow and refresh the watchlist (not a fixed shortlist)
`watchlist.json` is a **growing universe**, not overwritten from scratch each run:
- Use WebSearch to find today's high-growth candidates: top market movers/gainers, notable earnings beats, sector momentum (AI/semis, biotech catalysts, etc.), screened toward stocks priced low enough that $200 (or remaining cash) can meaningfully buy shares.
- **Add** newly-found promising names to `watchlist.candidates` (don't drop existing ones just because they weren't mentioned today).
- **Update** existing entries' notes/last-seen-price/thesis if you found fresher information on them today.
- It's fine and expected for this list to grow over the course of the 30-day challenge (aim to have accumulated a broad set — dozens of names across a few sectors — by the end, not just 3-5).
- Every open holding must always be present in the watchlist (so it keeps getting priced/reviewed each run even if it's no longer a "hot" candidate).

### 3. Get prices for anything you might act on
You don't need a fresh price for every single name in the whole accumulated watchlist every run — that won't scale. Prioritize: all open holdings (always), plus the strongest handful of candidates from today's refresh (step 2) that look buy-worthy. For each of those, get a price in this priority order:
- Try the TipRanks financial data MCP connector first, if available and not rate/quota-limited.
- Otherwise, fetch Alpha Vantage `GLOBAL_QUOTE` via the `WebFetch` tool (not `curl`/Bash — sandbox network egress policy blocks direct Bash internet access and may also block alphavantage.co specifically; if so, skip straight to the next option rather than retrying).
- **Fallback — WebSearch-sourced price**: if both of the above are unavailable, use a price found via WebSearch, but only if:
  - It comes from a specific, named, reputable source (e.g. a finance site's quote page, not a vague aggregator claim), and
  - You can cross-reference it against at least one other independent source and they roughly agree (within ~1-2%), and
  - You note in the recommendation's reasoning that the price is **web-search-sourced, not exchange-API-verified**, with the source and timestamp/date it reflects.
- If no price can be established even via WebSearch fallback for a ticker, exclude it from this run's buy/sell sizing.
- Because WebSearch-sourced prices can be stale or wrong, size positions a bit conservatively when relying on this fallback (e.g. leave a small cash buffer rather than spending 100% of available cash on unverified prices).

### 4. Produce a full buy list AND a full sell list (not just one recommendation)
- **Buy list**: every priced candidate you'd genuinely recommend buying right now, **ranked by expected profit potential** (highest-conviction / highest expected return first). For each: ticker, suggested qty, price, brief reasoning, and expected-return rationale (why this one and why that ranking). The suggested buys don't all have to be affordable simultaneously — rank them, and note in the email how far down the list current cash actually covers, so the user can act on however many they choose top-down.
- **Sell list**: check every open holding and include any that you think should be sold now (bad news, thesis broken, hit a target, better use of the capital elsewhere, etc.) — again respecting the no-same-day-round-trip rule. If a holding should be kept, don't list it.
- It's fine for either list to be empty on a given run if nothing clears the bar — don't force a trade.

### 5. Update `portfolio.json`
- For each open holding, append today's price to its `mark_to_market` array.
- Append this run's full recommendation set to `recommendations_log` as one entry: `{"timestamp", "buys": [...ranked list...], "sells": [...list...]}`. Do NOT change `cash`/`holdings` based on recommendations alone — those only change when the user reports back an actual executed trade (see below).

### 6. Send the email
Send an email (via the **AgentMail** MCP connector — the Gmail connector only supports creating drafts, not sending, so it must not be used for this step) to the address in `config.json` with:
- The ranked buy list and the sell list from step 4, including price source for each (TipRanks / Alpha Vantage / web-search fallback).
- Current cash, open holdings, and unrealized P&L per holding.
- Realized P&L to date and days remaining in the 30-day challenge.
- The not-financial-advice line.

## When the user reports an executed trade
This happens in a normal chat with the user (not a scheduled run) — the user says something like "I bought 2 shares of INTC at $91.50" or "I sold my IRWD position at $4.10". When that happens:
- **Buy**: add an entry to `holdings` (ticker, qty, buy_price, buy_date = today, empty `mark_to_market`), subtract cost from `cash`. Commit and push the updated `portfolio.json`.
- **Sell**: remove the matching entry from `holdings`, compute `days_held`, `profit` ($ and %), append it to `history` with `sell_price`/`sell_date`, add proceeds to `cash`. Refuse (and explain) if `buy_date` is today — same-day round trips aren't allowed. Commit and push.
- Always confirm back to the user what was recorded (qty, price, resulting cash balance) so they can catch mistakes.

# Stock Scanner Agent — Run Instructions

Project directory: `~/Coding/stock-scanner/`
Files: `portfolio.json` (state), `watchlist.json` (working candidate list), `config.json` (API key, email, schedule).

## Goal
Maximize total return on the $200 starting cash between `challenge_start` and `challenge_end` in `portfolio.json`. This favors decisive, higher-conviction moves over passive holding, especially as `challenge_end` approaches — but never at the cost of the hard rules below.

## Hard rules (never violate)
1. **No same-day round trip**: never recommend selling a ticker whose `buy_date` in `portfolio.json` equals today's date.
2. **Never recommend a buy that costs more than current `cash`** (qty × current price + nothing else, no margin/leverage).
3. Only recommend real, liquid, tradable US equities/ETFs — no options, no crypto, no leverage products unless explicitly reconsidered later.
4. This is not financial advice. Every email must say so briefly.

## Each run, do this in order
1. Read `portfolio.json` for current cash, open `holdings`, and `challenge_end`.
2. Source candidates: use WebSearch for today's top market movers, notable news/catalysts, and sector trends. Build a shortlist (a handful of names) biased toward stocks priced low enough that $200 (or remaining cash) can meaningfully buy shares.
3. For the shortlist **and** every open holding, get a price, in this priority order:
   - Try the TipRanks financial data MCP connector first, if available and not rate/quota-limited.
   - Otherwise, fetch Alpha Vantage `GLOBAL_QUOTE` via the `WebFetch` tool (not `curl`/Bash — sandbox network egress policy blocks direct Bash internet access and may also block alphavantage.co specifically; if so, skip straight to the next option rather than retrying).
   - **Fallback — WebSearch-sourced price**: if both of the above are unavailable, use a price found via WebSearch, but only if:
     - It comes from a specific, named, reputable source (e.g. a finance site's quote page, not a vague aggregator claim), and
     - You can cross-reference it against at least one other independent source and they roughly agree (within ~1-2%), and
     - You note in the recommendation's reasoning that the price is **web-search-sourced, not exchange-API-verified**, with the source and timestamp/date it reflects.
   - If no price can be established even via WebSearch fallback for a ticker, exclude it from this run's buy/sell sizing (hold is always safe to fall back to).
   - Because WebSearch-sourced prices can be stale or wrong, size positions a bit conservatively when relying on this fallback (e.g. leave a small cash buffer rather than spending 100% of available cash on an unverified price).
4. Decide one clear recommendation for this run: buy X shares of TICKER, sell TICKER, or hold/no action. Justify briefly with the news/data found. Respect the hard rules above.
5. Update `portfolio.json`:
   - For each open holding, append today's price to its `mark_to_market` array.
   - Append this run's recommendation (ticker, action, reasoning, price, timestamp) to `recommendations_log`. Do NOT change `cash`/`holdings` based on the recommendation alone — those only change when the user reports back an actual executed trade (see below).
6. Overwrite `watchlist.json` with today's candidate list + open holdings, and `last_updated`.
7. Send an email (via the **AgentMail** MCP connector — the Gmail connector only supports creating drafts, not sending, so it must not be used for this step) to the address in `config.json` with:
   - Today's recommendation and reasoning, **including which price source was used (TipRanks / Alpha Vantage / web-search fallback)** so the user knows how much to trust the price before acting.
   - Current cash, open holdings, and unrealized P&L per holding.
   - Realized P&L to date and days remaining in the 30-day challenge.
   - The not-financial-advice line.

## When the user reports an executed trade
In a normal chat (not a scheduled run), when the user says what they actually bought or sold:
- **Buy**: add an entry to `holdings` (ticker, qty, buy_price, buy_date = today, empty `mark_to_market`), subtract cost from `cash`.
- **Sell**: remove the matching entry from `holdings`, compute `days_held`, `profit` ($ and %), append it to `history` with `sell_price`/`sell_date`, add proceeds to `cash`. Refuse (and explain) if `buy_date` is today — same-day round trips aren't allowed.

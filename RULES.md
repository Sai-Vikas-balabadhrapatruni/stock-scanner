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
3. For the shortlist **and** every open holding, get a live quote:
   - Try the TipRanks financial data MCP connector first, if available and not rate/quota-limited.
   - Otherwise, fetch Alpha Vantage `GLOBAL_QUOTE` **via the WebFetch tool** (not `curl`/Bash — the sandbox's Bash has no direct internet access, only WebFetch/WebSearch/MCP connectors can reach the network): `WebFetch` the URL `https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol=TICKER&apikey=KEY` (key in `config.json`/prompt, ~25 requests/day cap — spend them only on names that survived step 2's filter). Optionally `TIME_SERIES_DAILY` for trend context on top candidates, same way.
   - If both fail for a ticker, do not guess a price from web-search snippets — treat it as unconfirmed and exclude it from this run's buy/sell sizing (hold is always safe to fall back to).
4. Decide one clear recommendation for this run: buy X shares of TICKER, sell TICKER, or hold/no action. Justify briefly with the news/data found. Respect the hard rules above.
5. Update `portfolio.json`:
   - For each open holding, append today's price to its `mark_to_market` array.
   - Append this run's recommendation (ticker, action, reasoning, price, timestamp) to `recommendations_log`. Do NOT change `cash`/`holdings` based on the recommendation alone — those only change when the user reports back an actual executed trade (see below).
6. Overwrite `watchlist.json` with today's candidate list + open holdings, and `last_updated`.
7. Send an email (via the Gmail connector) to the address in `config.json` with:
   - Today's recommendation and reasoning.
   - Current cash, open holdings, and unrealized P&L per holding.
   - Realized P&L to date and days remaining in the 30-day challenge.
   - The not-financial-advice line.

## When the user reports an executed trade
In a normal chat (not a scheduled run), when the user says what they actually bought or sold:
- **Buy**: add an entry to `holdings` (ticker, qty, buy_price, buy_date = today, empty `mark_to_market`), subtract cost from `cash`.
- **Sell**: remove the matching entry from `holdings`, compute `days_held`, `profit` ($ and %), append it to `history` with `sell_price`/`sell_date`, add proceeds to `cash`. Refuse (and explain) if `buy_date` is today — same-day round trips aren't allowed.

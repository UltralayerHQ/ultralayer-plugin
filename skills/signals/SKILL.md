---
name: signals
description: How the market is acting toward a company, a sector, an asset class, a country or the whole US market, with history, such as volatility, beta, abnormal moves, news attention and sentiment, insider buying, market breadth, and market regime. Operations are list_signals, retrieve_signal, retrieve_signal_cross_section and retrieve_signal_coverage.
---

# Signals

Signals tell you how the market is acting toward a company, an industry, a sector, an asset class, a country or the market as a whole, for example how a stock moves and why, what the news says, what insiders do, and what state the market is in. Each comes with its history, and company signals are on one scale for every company, so readings can be compared, ranked, screened and followed over time.

| Operation | Role |
|-----------|------|
| `list_signals` | The catalog: what each signal tells you, how to read it, and whether the user's plan includes it |
| `retrieve_signal` | One signal for one company or group, over time |
| `retrieve_signal_cross_section` | One signal for many companies or groups at once, to rank, screen or compare |
| `retrieve_signal_coverage` | The companies or groups a signal covers |

`list_signals`, `retrieve_signal_coverage` and every refusal are free.

**Plan:** Standard. Some signals and `retrieve_signal_cross_section` need Pro; each signal's `on_your_plan` says whether the user's plan includes it.

---

## The signals

`list_signals` gives the current list and each signal's details.

- **How a stock moves:** `realized_volatility` (past month), `volatility_forecast` (coming month), `own_volatility` (the company's own part), `market_beta`, `variance_shares` (market, sector and own shares of its movement), `abnormal_move` (how unusual the day's own move was).
- **What the news says:** `attention` and `company_sentiment` for a company. For a sector, an asset class or a country: `scope_sentiment` (the day's news in total) and `scope_tone` (the average item, comparable between groups).
- **What insiders do:** `insider_cluster` (several insiders buying), `insider_net_buying` (six months), `insider_net_buying_month`.
- **The state of the market:** `volatility_regime` (calm, normal, stressed), `trend_regime` (above or below trend, correction, bear market), `breadth` (how much of the market takes part).

---

## When to use

**Use it for** any question about how the market is acting or reacting, for example whether a move was the company's own, how much a stock moves and could move, how a market fall would hit the user's holdings, whether news about a company or a sector is building or turning, what insiders are doing, and what kind of market this is. Also to rank a watchlist, screen every covered company, or test an idea on what was known at the time.

**Not this tool:** prices and fundamentals → a market-data source. The story behind a reading → Wire and developments. A company's outlook → `list_guidance`. The open web → your own web search.

---

## Use cases

**Did the stock move on its own news?** A large `abnormal_move` is the company's own; near 0, it moved with the market.
```json
{ "signal": "abnormal_move", "symbol": "NVDA", "limit": 5, "detail": "standard" }
```

**If the market falls 10%, how much will my stocks fall?** A beta of 2 is about twice the market's fall.
```json
{ "signal": "market_beta", "symbols": ["NVDA", "TSLA", "KO", "BRK.B"], "detail": "essential" }
```

**Which of my stocks is getting more news than usual?** Each is against its own usual. Read `attention_week`.
```json
{ "signal": "attention", "symbols": ["NVDA", "PLTR", "KO"], "detail": "essential" }
```

**Which companies' insiders bought more than they sold last month?** Every covered company in one call. Keep the positive readings: buying is rare.
```json
{ "signal": "insider_net_buying_month", "detail": "essential" }
```

**Is the news about a sector turning?** The sector's average news tone against its own usual: above 0 is warmer.
```json
{ "signal": "scope_tone", "group_kind": "wire_sector", "group_id": "energy", "limit": 30, "detail": "standard" }
```

**Is the market weakening under the surface?** Fewer stocks above trend (`share_above`) means a narrowing base. One group, so no key.
```json
{ "signal": "breadth", "limit": 60, "detail": "standard" }
```

**As it was known then.** A past range, with each row's `timestamp`, for a backtest.
```json
{ "signal": "market_beta", "symbol": "META", "start_period": "2026-06-01T12:00:00Z", "end_period": "2026-06-09T12:00:00Z", "detail": "full" }
```

---

## Response detail

Default is `full`. `detail` only drops fields. Price does not change.

| Level | `retrieve_signal`, `retrieve_signal_cross_section` | `list_signals` |
|-------|------------|----------------|
| `essential` | The readings and `units` | Name, title, summary, plan, `on_your_plan` |
| `standard` | That plus the signal's summary | That plus use cases, key kind, group kinds, cadence, `history_from`, columns, units |
| `full` | That plus what each column means, and each row's `timestamp` | That plus `reading`, `validation`, `limits` |

Choose a signal from `list_signals` at `standard`. Before you interpret one, read its entry at `full` (`signals: ["name"]`): `reading` says how to read a value, and `limits` says when it misleads. Name a signal to the user by its `title`.

---

## What you get back

| Field | Notes |
|-------|-------|
| `symbol` / `group_kind` / `group_id` | The key you asked for; null for the other kind |
| `units` | One per column, at every level. **Read it before you quote a number.** |
| `rows[].period` | The instant the measured day or month ended, UTC. See **Periods**. |
| `rows[].timestamp` | When Ultralayer had the reading. `full` only |
| `rows[].values` | Column name → value |
| `rows[].readings[]` | Cross-section only: one per company (`symbol`) or group (`group_kind`, `group_id`), each with `values` |
| `unmatched_symbols` | Cross-section only: the `symbols` that are not covered. Tell the user which are missing. |

Rows are oldest first. `retrieve_signal` returns the newest `limit` readings in the range (default 14, max 1000), and the cross-section the newest `limit` periods (default 1, max 30).

`retrieve_signal_coverage` returns a company signal's covered companies (`symbol`, `name`) and the rule that selects them, or a group signal's groups.

---

## Company or group

The entry's `key_kind` says which.

- **`security`**: give `symbol`. Coverage is several hundred of the largest US-listed companies, with one share class each: Alphabet is `GOOGL`, not `GOOG`, and Berkshire is `BRK.B`. Case does not matter. A company outside coverage is refused, free, with `not_covered`, so you do not need to check first.
- **`group`**: give `group_kind` and `group_id`, in the signal's own words. The entry's `group_kinds` names the kinds, and `retrieve_signal_coverage` lists the groups. A sector's kind is `wire_sector`: technology is `wire_sector`, `information_technology`. A country is its two-letter code: the United Kingdom is `gb`. A signal with one group needs neither parameter. A group cross-section returns every group of the signal.
- **An industry** has no group: take a cross-section of its companies with `symbols`.

---

## Periods

`period` is the instant the measured day or month ended, so its date is often the next day: `2026-11-24T05:00:00Z` is the reading for Monday, November 23. Each signal's `cadence` says where its periods end, with an example. Tell the user the day or month that was measured.

`start_period` and `end_period` filter on `period`, and both ends are included. For the days D through E, ask from noon UTC on D to noon UTC on the day after E. For November 23 alone, that is `2026-11-23T12:00:00Z` to `2026-11-24T12:00:00Z`. With neither, you get the newest readings.

`history_from` is the signal's first reading. The user's plan reaches back a fixed length from today, which can be later. A `start_period` before that is refused with `history_limit`, and the message gives the earliest date you can ask for.

For a backtest, use only readings whose `timestamp` is at or before the simulated moment. On history from before a signal went live, `timestamp` is a conservative estimate. A symbol means the company that held it at the end of your range. Give `version` to keep a backtest on one way of computing a signal; without it you get the newest.

---

## Workflow

**Explain a move**
1. `abnormal_move`: was the move the company's own?
2. `attention` over the same days. A large move on usual news points to its peers or its theme.
3. `list_wire` or `search_developments` for what happened.

**The company, or its industry?**
1. `company_sentiment` cross-section with the company and its peers as `symbols`, `limit` up to 30.
2. The company falling alone is its own news; falling together is the industry's.
3. `list_wire` and `list_guidance` for the cause.

**Read the market**
1. `trend_regime` and `volatility_regime` for its state.
2. `breadth` for whether most stocks or a few large ones carry it.
3. `scope_tone` cross-section, reading the `wire_sector` groups, for whether bad news is in one sector or has spread.

---

## Combine with

- **Wire and developments**: what happened and why, with sources.
- **Guidance**: a company's outlook beside how its stock and news react.
- **`screen_stocks`** and **`identify_stakeholders`**: the companies exposed to a theme, then signals for how the market treats each.
- **`entity_co_mentions`**: what a company is in the news with, beside `attention`.

---

## Limitations

- Signals are daily or monthly. The newest reading is for the last finished trading day or month, never the session in progress.
- Readings exist only for US trading days. A missing day is not a zero.
- Coverage is a defined set of companies, not every listing. Funds and ADRs are not covered.
- A cross-section without `symbols` returns every covered company, several hundred readings per period.

---

## FAQ

**Does a signal predict future returns or price moves?**
No. It tells you how the market is acting and reacting. `volatility_forecast` forecasts the size of next month's moves, not their direction.

**The reading is dated a day late.**
`period` is when the measured day ended: midnight New York after the session is the next day in UTC. The `cadence` names the day.

**Two volatility numbers disagree by a factor of 100.**
A company's volatility is an annualized fraction (0.46 is 46% a year), and the market's `realized` is in annualized percent (20 is 20% a year). `units` says which.

**`SPY` is refused as `unknown_symbol`.**
A fund is not a company. For the market, use a signal about the `market` group.

**`GOOG` is `not_covered`.**
Each company is covered under one share class: use `GOOGL`.

**Is a cross-section worth it for a few companies?**
For the latest reading of more than five companies, one cross-section costs less than calling `retrieve_signal` for each.

**Do I need `retrieve_signal_coverage` to check one company?**
No. Ask for the company; a refusal is free. Call coverage when the user wants the list.

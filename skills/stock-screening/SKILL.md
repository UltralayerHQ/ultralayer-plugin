---
name: stock-screening
description: Screen public companies by exposure to a theme, event or trend described in plain language, combined with ordinary screener filters (country, market cap, valuation, growth, performance). Operation is screen_stocks.
---

# Stock screening

`screen_stocks` answers “which listed companies are most exposed to X?” One plain-language query carries both the exposure theme and any ordinary screener constraints. The exposure part is answered from Ultralayer's stored, cited company-impact evidence; the constraints run as hard filters on current listing data.

| Operation | Role |
|-----------|------|
| `screen_stocks` | Plain-language criterion → ranked companies with exposure score, exposure type, rationale, sources |

**Latency:** typically 20–30 seconds.

---

## When to use

**Use when you need:**
- A ranked list of tickers tied to a theme, trend, policy or event (beneficiaries, losers, or both)
- Theme exposure combined with screener constraints (“US, under $10bn, exposed to …”)
- Small and mid-cap or non-US names a generic model would not surface, each with a cited reason
- A fast, cheap first pass before deeper work on individual names

**Do not use when you need:**
- A fresh, deep assessment of one concrete situation with a factor matrix, or a point-in-time backtest → `identify_stakeholders`
- A pure fundamentals screen with no theme (“P/E under 15 and growth over 20%”) → rejected with a 422 and not charged; see **Queries with no theme**
- One company's dossier → `retrieve_entity`
- Headlines or the story behind a move → Wire
- Prices, filings, open-web research → your native web search tool

---

## Use cases

**Broad theme, no constraints** — A short open theme works well: the screen fans it out into the distinct ways a company can be tied to it.
```json
{
  "query": "Companies going to be replaced by AI",
  "detail": "standard"
}
```

**Whole supply chain** — Name a technology or product and ask for its supply chain to get suppliers, equipment makers and enablers, not only the obvious names.
```json
{
  "query": "Optical computing supply chain",
  "limit": 40,
  "detail": "standard"
}
```

**Theme with size and country constraints** — Find the smaller names behind a theme that large-cap lists miss.
```json
{
  "query": "US companies under $10bn that benefit from AI data centre power demand",
  "detail": "full"
}
```

**Winners and losers** — Ask for both sides to get a signed map of a trend in one call.
```json
{
  "query": "winners and losers from GLP-1 weight loss drugs",
  "limit": 40,
  "detail": "standard"
}
```

**One-sided shock map** — Ask for only the hurt (or only the helped) side of an event.
```json
{
  "query": "companies hurt by a Strait of Hormuz closure",
  "detail": "essential"
}
```

**Regional exposure** — Restrict by country and let the theme define the industry.
```json
{
  "query": "Japanese and Taiwanese companies exposed to US chip export controls on China",
  "detail": "full"
}
```

**Supply chain of one company** — A bare company name or ticker is read as a theme: who is tied to it.
```json
{
  "query": "suppliers most dependent on NVIDIA accelerator demand"
}
```

**Layered constraints** — Mix numeric filters with qualitative ones; the qualitative part is judged per company, not filtered.
```json
{
  "query": "Indian mid caps between $1bn and $10bn with net margin above 10% that benefit from rising defence spending, not state-owned",
  "detail": "full"
}
```

---

## Writing the query

`query` (1–2000 chars) is the only steering input. The whole request is rewritten into a plan before anything is screened, and every part of the query lands in one of two places:

| Part of the query | Becomes | Effect |
|-------------------|---------|--------|
| Theme, event, trend, direction (“benefit from”, “hurt by”) | `criterion` + score rubric + exposure types | Judged per company from evidence |
| Country, exchange, sector/industry, market cap, P/E, growth, margin, leverage, performance, liquidity | `quant_filters` | Hard pass/fail before the exposure screen |

What follows from that:

- **Direction words set the sign range.** “benefit from” returns scores 0 to 1; “hurt by” returns negatives only; “winners and losers” or “exposed to” returns both.
- **An industry word becomes a hard filter.** “Semiconductor companies in Japan or Taiwan exposed to …” filters on the market-data industries that cover semiconductors before any evidence is read. That taxonomy often differs from intuition (Tokyo Electron is `Producer Manufacturing / Industrial Machinery`; Apple is `Electronic Technology / Telecommunications Equipment`), so check `quant_filters`. If an expected name is missing or the list is thin, move the industry into the theme: “Japanese and Taiwanese companies exposed to US chip export controls”.
- **Vague words become invented filters.** “Large cap”, “mid cap”, “liquid”, “high dividend” are converted to thresholds the planner picks. State numbers yourself when they matter. Quality words alone (“good stocks”) name no theme and are rejected; see **Queries with no theme**.
- **A list of tickers is not a universe filter.** “Among AAPL, MSFT, GOOGL, AMZN and META …” became `market_cap > $1tn`. There is no symbol filter; screen the theme and pick out the names you care about.
- **Qualitative conditions are not filters.** “not state-owned”, “pure play”, “with a signed contract” go into the criterion and are applied by judgment.
- **Dates in the query do not make the screen point-in-time.** “As of December 2021” is copied into the criterion, but evidence and market data are current.
- **“Sorted by …” does not order the results.** See `quant_sort` below.

Filterable fields: `country`, `exchange`, `sector`, `industry`, `market_cap_basic` (USD), `price_earnings_ttm`, `dividends_yield_current` (%), `total_revenue_yoy_growth_ttm` (%), `net_margin_ttm` (%), `debt_to_equity`, `Perf.1M` / `Perf.YTD` / `Perf.Y` (price change %), `Value.Traded` (USD, latest session). Anything else (EV/EBITDA, ROE, free cash flow, analyst ratings) cannot be a hard filter.

---

## Response detail

`detail` **defaults to `full`**. Reduced levels drop keys only; surviving fields keep the same names and paths.

| Level | What you get |
|-------|--------------|
| `essential` | `results[]` with name, symbol, `exposure_score`, `exposure_type`, `rationale` |
| `standard` | essential + `criterion` + `quant_filters` + `quant_filter_match_count` + citable sources (URL, timestamp; no quotes) |
| `full` | standard + source quotes, `score_rubric`, `exposure_types`, `quant_sort`, per-company `market_data` |

**Prefer `standard`.** It shows which filters were actually applied and how many companies passed them, which is enough to diagnose an empty or odd result. **Use `full`** when you need per-result market data (to sort or filter yourself), source quotes, the score rubric or `quant_sort`.

---

## What you get back

### Plan (top level)
- `criterion`: your query restated as one sentence. Read it first; it is what was actually screened.
- `score_rubric`: what sign and magnitude mean for this criterion. Written per call, so scores are comparable within one response, not across calls.
- `exposure_types[]`: the categories companies are grouped into for this screen.
- `quant_filters[]` (`standard` and `full`): `{field, operation, values}` with `operation` in `equal` (any of the values) / `greater` / `less` / `in_range`. Empty when the query had no screener constraints.
- `quant_filter_match_count` (`standard` and `full`): companies that passed the filters; `null` when no filter ran.
- `quant_sort`: which companies are kept when more pass the filters than can be screened (default: largest market cap first). It selects the candidate pool; it does **not** order `results`. A request for “small caps” with no upper bound will still favour the largest names that pass.

### `results[]`
- `stakeholder`: `{name, symbol}` (+ `stakeholder_type: "PUBLIC_COMPANY"` in `full`). Non-US symbols carry listing suffixes (`8035.T`, `051910.KS`, `RELIANCE.NS`, `0386.HK`).
- `exposure_score` ∈ [-1, 1]. Positive = benefits as the criterion strengthens, negative = hurt. |score| ≥ 0.5 is strong, direct exposure; below 0.5 is secondary or diversified.
- `exposure_type`: usually one of `exposure_types`, occasionally a new label when none fit.
- `rationale`: one or two sentences, backed by the cited sources.
- `sources_with_quotes[]`: `url`, `source_timestamp`, and (in `full`) exact `quotes`. Check `source_timestamp` to judge how fresh the evidence is.
- `market_data` (`full` only; `null` when no filter ran): country, sector, industry, exchange, market cap, P/E, growth, margin, performance, value traded, debt/equity. Current values, usable for your own sort or second-pass filter.

Results are ordered by absolute `exposure_score`, strongest first, so winners and losers are interleaved.

`limit` (default 20, max 40) is a ceiling. Fewer come back when evidence is thin or the filters are narrow.

---

## Empty results

`results: []` is a normal 200 response, not an error. At `detail: "standard"` or `"full"`, read `quant_filter_match_count`:

| `quant_filter_match_count` | Meaning | Fix |
|----------------------------|---------|-----|
| `0` | The filters alone match no company | Relax or drop a filter |
| Small (tens) | Companies pass, but none has stored evidence on the theme | Widen the numeric bounds, or drop them and filter on `market_data` yourself |
| Large, still empty | A sector/industry filter excluded the relevant companies, or a filter was repeated as a condition the evidence cannot confirm | Move the industry into the theme wording; drop the filter and apply it yourself on `market_data` |
| `null` | No filter ran; no evidence found for the theme | Rephrase around the mechanism (what changes, who is affected), or use `identify_stakeholders` for a fresh assessment |

---

## Queries with no theme

A query that holds screener filters only (“P/E under 15 and revenue growth over 20%”, “high dividend US small caps”, “good stocks”) is rejected within a few seconds with a 422, error `no_exposure_theme`, and is not charged. Every result must cite stored evidence of exposure to something, so a filters-only query has nothing to rank on.

Fix: add what the companies should be exposed to (“high dividend US utilities that benefit from data centre power demand”). A named line of business counts as a theme, so “Japanese biotech companies over $5bn” runs.

---

## Workflow

| User intent | Approach |
|-------------|----------|
| “Which stocks are exposed to X?” | Theme-only query, `standard` |
| “Small caps / in country Y exposed to X” | Theme + explicit numeric bounds, `standard`; verify `quant_filters` |
| “Winners and losers from X” | Say both sides in the query; raise `limit` |
| “Rank them by performance / valuation” | Screen with `full`, then sort `market_data` yourself |
| “Is ticker T exposed to X?” | Screen X and look for T; absence is not evidence of no exposure |
| “Deep dive on a specific event, or as of a past date” | `identify_stakeholders` |

1. Write one sentence: mechanism + direction + only the constraints that matter, with explicit numbers.
2. Call once. Use `standard`, or `full` if you need market data or quotes.
3. Read `criterion` and `quant_filters` to confirm the screen is the one intended. If not, reword and call again.
4. Brief from the top of `results`: score, exposure type, rationale, one quote for key names.
5. Follow up on chosen symbols with `retrieve_entity`, `search_developments(symbols=…)`, Wire, or your native web search tool.

---

## Limitations

- Public companies only. Coverage follows stored evidence: a company with no evidence on the theme is absent however exposed it is, so the list is not exhaustive.
- Not deterministic. Repeating a query changes exposure-type labels, scores (by 0.1–0.2 in places) and part of the list. The strongest names recur.
- No point-in-time mode and no date parameter. Evidence is whatever is stored now; market data is current.
- No live web search in the call. Very recent events may be thinly covered.
- When very many companies pass the filters, only the top slice by `quant_sort` is screened.
- The ten numeric fields above are the only hard filters. No symbol list, no custom fields.
- Scores are reasoned exposure estimates against a per-call rubric, not price targets or trading signals.

---

## FAQ

**Q: How is this different from `identify_stakeholders`?**
A: Screening is fast, takes screener filters, and reads stored evidence across many past assessments. `identify_stakeholders` runs a fresh ~2 minute assessment of one situation with live retrieval, a factor matrix, steering inputs and backtest mode.

**Q: I asked for results sorted by 1-month performance and they are not.**
A: Results are always ordered by absolute exposure score. Sort on `market_data` in a `full` response.

**Q: Why did I get fewer than `limit`?**
A: `limit` is a ceiling. Only companies with supporting evidence that also pass every filter are returned.

**Q: Why is `market_data` null?**
A: No screener filter ran. Add a constraint (for example a country or a market-cap bound) to get it.

**Q: Can I screen on fundamentals alone?**
A: No. A query with filters only and no theme is rejected with a 422 (`no_exposure_theme`) and not charged. Add a theme; the filters then narrow the universe it is screened over.

**Q: Why did a well-known company in the sector not appear?**
A: Either a sector/industry filter excluded it (check `quant_filters`), or it fell outside the candidate pool or the evidence retrieved. Reword with the industry in the theme, or raise `limit`.

**Q: Empty or over-long query?**
A: 422. `query` must be 1–2000 characters and not blank.

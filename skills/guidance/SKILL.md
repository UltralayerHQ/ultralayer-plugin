---
name: guidance
description: Official company outlooks, how they revised, how they compared to the reported actual, and the aggregate score of that. Operations are list_guidance, list_guidance_outcomes, and guidance_bias.
---

# Guidance

`list_guidance` is the official outlook tape: what the company said it expects, when it said it, and how that number moved vs the last observation of the same series.

`list_guidance_outcomes` is company guidance versus the reported actual. One pair per series after that period reported.

`guidance_bias` is the aggregate score of those results. One point per report period.

| Operation | Role |
|-----------|------|
| `list_guidance` | Current and past outlooks, with revision history |
| `list_guidance_outcomes` | Company guidance versus the reported actual |
| `guidance_bias` | Average of those results, one score per report |

Every filter works **without a ticker list**, so these are cross-sectional screens as much as single-name lookups: who moved an outlook today, which peers guide the same period, who missed their own guide this cycle. `end_timestamp` rebuilds any of it as of a past day, so a screen can be backtested.

---

## When to use

**`list_guidance`** — What they expect now, who just guided, who raised or cut, the history of one line, guidance for the same metric across companies, or the book as of a past day.

**`list_guidance_outcomes`** — What they guided versus what they reported. Empty until that period reports; the live outlook stays on `list_guidance`.

**`guidance_bias`** — Whether a name habitually guides below what it prints, which is a prior on the next guide.

**Not these tools:** sell-side consensus, rumor, or reaction → Wire / developments / your own web search. Wording changes in the release → `list_changes`. A company dossier → `retrieve_entity`.

---

## Detail

API default is `full`. **Set `standard` unless you need the numbers.** `detail` does not change price. `guidance_bias` has no `detail`.

**`list_guidance`**

| Level | What you get |
|-------|----------------|
| `standard` | Quote, `revision_direction`, `source_type`, cite. What an LLM should read. |
| `full` | Parsed numbers (ranges, YoY, `revision_midpoint_delta_pct`), `observation_count`, `cik`, `source_metadata` |
| `essential` | Quote, identity, `source_type`, `revision_direction`. No company name, ingest timestamp, URL, or numbers. |

**`list_guidance_outcomes`**

| Level | What you get |
|-------|----------------|
| `standard` | Quotes, labels, cites |
| `full` | Guided range, actual, deltas, `rationale`, `cik` |
| `essential` | Identity and outcome label |

---

## Clocks and shared filters

Time filters use when **we recorded** the row, not publication time. `end_timestamp: null` = now. A future end is treated as now.

| Tool | `start_timestamp` / `end_timestamp` |
|------|--------------------------------------|
| `list_guidance` | When the outlook was recorded. `start` only picks which series appear; history inside a series goes up to `end` |
| `list_guidance_outcomes` / `guidance_bias` | When the **result** was recorded, not when they guided |

For “that quarter,” use `guided_period_end_from` / `_to` or `guided_fiscal_year` / `guided_fiscal_quarter`. A May print of Q1 is not an Apr–Jun window. `guided_fiscal_quarter=4` is the Q4 outlook, not every pair that printed in Q4.

AND across fields; OR inside a list. Same names on `list_guidance` and `list_guidance_outcomes` where the field exists.

| Param | Notes |
|-------|-------|
| `symbols` | Tickers, max 8 |
| `metric_keys` | Substring, case does not matter. `revenue` also matches `revenue_growth`. Pin `metric_key_exact_match: true` for one key |
| `metric_basis` | `gaap` / `non_gaap` / `unspecified`. One sentence is often two series (GAAP and non-GAAP) — do not double-count |
| `segments` | Named cuts, or `"null"` for company-wide. `"consolidated"` will miss |
| `guided_period_end_*` | Calendar day the outlook period ends — use this to line up companies |

`guidance_bias` is one `symbol` and an optional window. No metric filter, no `limit`, no `detail`.

---

## `list_guidance`

One object per series: ticker × metric × GAAP/non-GAAP × segment × period. `values[]` is that line’s history, newest first.

Default `limit` is 10 (max 100). Default `values_mode` is `all`. Pin a metric or raise `limit` so one name does not fill the page.

`latest` is the newest observation **per series** (often the call), not “current slide only.” For the **printed number**, use `values_mode: "all"` and take the latest `earnings_release` row in `values[]`. If the call and an 8-K disagree, still use that release row. Keep the call for color.

### Use cases

**Realtime guidance tape** — Every series that just moved, market-wide, minutes after the print. No tickers. This is the desk feed.
```json
{
  "values_mode": "latest",
  "sort": "timestamp",
  "detail": "standard",
  "limit": 40
}
```

**Who raised or cut, market-wide** — Cross-sectional revision screen, no ticker list. `revision_delta` puts the biggest midpoint moves first. Read `metric_key` before calling a raise good: OpEx, tax, and cost guides go up on bad news. Sanity-check the top rows against the quote — an enormous `revision_midpoint_delta_pct` usually means the two observations were stated at different scales, not that the outlook moved that much.
```json
{
  "revision_direction": ["raised", "lowered"],
  "values_mode": "latest",
  "sort": "revision_delta",
  "detail": "full",
  "limit": 40
}
```

**Peer cohort on one metric** — Same metric, same calendar period-end, ranked like-for-like. Fiscal labels do not align across companies; `guided_period_end_*` does. `comparable` ranks on the more common comparison in the match set (growth % vs rate level); rows without that number sort last.
```json
{
  "metric_keys": ["revenue"],
  "metric_key_exact_match": true,
  "guided_period_end_from": "2026-12-01",
  "guided_period_end_to": "2027-01-31",
  "values_mode": "latest",
  "sort": "comparable",
  "detail": "full",
  "limit": 40
}
```

**One company’s outlook pack** — Every open line. Split rows by quarter vs year on `guided_period_type` / `guided_period_end_date`.
```json
{
  "symbols": ["NVDA"],
  "values_mode": "latest",
  "detail": "standard",
  "limit": 40
}
```

**The number they printed** — `latest` can be the call; the release row is the citable figure. History keeps both.
```json
{
  "symbols": ["NVDA"],
  "metric_keys": ["revenue"],
  "metric_key_exact_match": true,
  "values_mode": "all",
  "detail": "full"
}
```

**Off-cycle revisions** — An 8-K that moves the outlook away from an earnings date. Rare and high-signal. `values[]` still holds the release it changed; pick rows by `values[].source_type`.
```json
{
  "source_types": ["guidance_update"],
  "values_mode": "all",
  "sort": "timestamp",
  "detail": "standard",
  "limit": 20
}
```

**The book as of a past day** — PIT. What the outlook was on date D, no later leakage. This is how a guidance signal gets backtested.
```json
{
  "end_timestamp": "2026-06-30T20:00:00Z",
  "values_mode": "latest",
  "sort": "timestamp",
  "detail": "standard",
  "limit": 40
}
```

### What you get back

| Field | Notes |
|-------|-------|
| `values[]` | Newest first |
| `guided_period_end_date` | Use this to line up peers |

**At `standard`, each value:** `quote`, `timestamp`, `source_type`, `revision_direction`, `source`. Trust the quote if a label disagrees.

`revision_direction` is vs the last observation of **this** series (`new` / `raised` / `lowered` / `maintained` / `withdrawn`), not good/bad. It can be `null` — that is not `new`.

**Full only:** `revision_midpoint_delta_pct` (percent of the prior midpoint: 4.0 → 4.5 is +12.5, not +0.5), ranges, YoY, `observation_count` (length as of `end_timestamp`, not always `len(values)`), `cik`, `source_metadata`.

**Source:** `earnings_release` / `guidance_update` / `earnings_call`. At `standard`, `source_metadata` is null; the URL is still on the source. At `full`, `source_metadata.fiscal_year` / `fiscal_quarter` are the **document’s** completed period (a Q2 release guiding Q3 has Q2 here). To join `list_changes`, bump this call to `full` and use those fields — not `guided_*`.

### Filters and sort

`revision_direction` matches the **latest** observation only.

`source_types` includes a series if **any** observation has that class. `values[]` stays the full history. `earnings_release` + `latest` can still return a later **call**. Read `values[].source_type`.

| `sort` | Order |
|-------|--------|
| `timestamp` (default) | Most recently updated series first |
| `guided_period_end` | Period-end, oldest first |
| `comparable` | Strongest like-for-like number in the match set (growth % or rate level, whichever is more common). Pin period-end dates for a fair cohort |
| `revision_delta` | Largest latest midpoint move |

---

## `list_guidance_outcomes`

Company guidance versus the reported actual. One pair per series after that period reported.

This is an estimate-vs-actual record where the estimate is **the company’s own**, not the street’s — scored, quoted, and cited. Run it without tickers for a market-wide beat/miss screen.

If the same outlook is on the press release and the next-day call, the release is kept. Then the newest remaining guide against that actual is kept. A preannouncement and a later full release stay two rows.

Default `outcomes` is the scored results only. Pass `indeterminate` to see rows that could not be scored. Omit does not mean all rows.

Default `limit` is 10. Raise it — 10 is a page, not the whole print.

### Use cases

**One name’s track record** — Every guide they set and what printed against it. The company’s own estimate table.
```json
{
  "symbols": ["AMZN"],
  "detail": "standard",
  "limit": 40
}
```

**Market-wide beat/miss screen** — No tickers. Who blew through or fell short of their own guide this cycle. `outcome_abs` puts the extremes first.
```json
{
  "outcomes": ["significantly_exceeded", "significantly_missed"],
  "sort": "outcome_abs",
  "detail": "standard",
  "limit": 40
}
```

**Read-across cohort** — One metric, one calendar period-end, across peers. Names that already reported give the base rate for names that have not. Reporting dates are staggered, so this window is where the lag lives.
```json
{
  "metric_keys": ["revenue"],
  "metric_key_exact_match": true,
  "guided_period_end_from": "2026-06-01",
  "guided_period_end_to": "2026-07-31",
  "detail": "full",
  "limit": 40
}
```

**That period for one name** — Pin the outlook period, not `start_timestamp`.
```json
{
  "symbols": ["NVDA"],
  "guided_period_end_from": "2026-07-01",
  "guided_period_end_to": "2026-08-31",
  "detail": "standard"
}
```

**Why a label looks wrong** — `indeterminate` rows are guides with no comparable actual (a segment cost call, a capacity target). `full` carries the `rationale`.
```json
{
  "symbols": ["AMZN"],
  "outcomes": ["indeterminate"],
  "detail": "full",
  "limit": 20
}
```

### What you get back

| Field | Notes |
|-------|-------|
| `outcome` | Vs **this** guide, not good/bad for the stock. Read the metric and the quotes |
| `guidance_quote` / `actual_quote` | Ground truth. Trust these over `outcome` and over `rationale`, which can disagree with each other |
| `actual_source_type` | `earnings_release` or `earnings_preannouncement` |

`standard` has the quotes and links. `full` adds the guided range, the actual, the deltas, and `rationale` (why that label).

Match to `list_guidance` on ticker + metric + GAAP/non-GAAP + segment + period. There is no shared id.

| `sort` | Order |
|-------|--------|
| `timestamp` (default) | Newest result first |
| `outcome_abs` | Largest miss or beat first (by label, not by dollars) |

---

## `guidance_bias`

One name. Oldest report first. The whole series in the window.

The pattern is the point, and it is **forward-looking**: a company that scores positive report after report guides low and beats, so its next guide is probably a floor. Persistently negative means the guide runs optimistic and should be discounted. A single point is a scorecard; the run is a prior on the next print.

```
significantly_exceeded = +2
exceeded               = +1
in_line                =  0
missed                 = -1
significantly_missed   = -2
```

Average of those scores. Unscored rows are left out.

Each point: `bias_score`, `resolved_count`, `label_counts`, and `fiscal_year` / `fiscal_quarter` of the **report that printed the actual**, not the guided period. Several lines can land in one report (revenue and operating income, GAAP and non-GAAP, a quarter guide and a year guide). They share one point. `resolved_count` of 2 is normal. The latest point is the last report, not the next print.

The outcomes list keeps only the newest guide per line. Bias averages every guide that closed that night, so `resolved_count` will not always match the outcomes list.

Do not rank many names from this tool (one symbol per call). Do not treat a single vote as a pattern.

### Use cases

**Sandbagger or over-promiser** — Read the run of `bias_score`, then pull the pairs behind it with `list_guidance_outcomes` on the same ticker.
```json
{
  "symbol": "AMZN"
}
```

**As of a past day** — PIT the score so a backtest only sees reports that had already printed.
```json
{
  "symbol": "AAPL",
  "end_timestamp": "2026-08-01T00:00:00Z"
}
```

---

## Workflow

Chains worth more than a single call.

**Earnings-night brief on one name**
1. `list_guidance` — ticker, `values_mode: "all"`, `full`. Printed figures come from the latest `earnings_release` row per series; the call row is color.
2. `list_guidance_outcomes` — same ticker. The period that just closed is now scored against the guide they gave a quarter ago.
3. `list_changes` — ticker + the fiscal year/quarter from step 1’s `source_metadata`. This is how the wording moved around those numbers: non-GAAP definitions, segment cuts, outlook assumptions.
4. Wire (legal name, `Nvidia Corporation`) for how the tape read it; your own web search for sell-side reaction.

**Read-across before the laggard prints**
1. `list_guidance_outcomes` — one metric + a `guided_period_end_*` window covering the cohort. Peers that already reported show whether guides for that period ran hot or cold.
2. `list_guidance` — same metric and window, `values_mode: "latest"`. The names still open, with the guide they are about to be measured against.
3. `guidance_bias` — on each open name. A habitual sandbagger into a hot cohort is a different setup than an over-promiser into a cold one.
4. `identify_stakeholders` or `search_developments` — when the driver is a shared shock (tariffs, memory pricing, freight rates) rather than a company story.

**Cross-sectional revision screen**
1. `list_guidance` — no tickers, `revision_direction: ["raised","lowered"]`, `sort: "revision_delta"`, `full`. The largest outlook moves in the market.
2. Split the results yourself on `metric_key` and `metric_basis`. A raised cost guide is not a raised revenue guide, and one sentence can be both GAAP and non-GAAP.
3. `list_changes` — `guidance_basis` on the same names, to separate real demand from a re-based outlook (a deal folded in, a tariff assumption added).
4. `market_signal` or Wire — whether the tape has already priced it.

**Backtest a guidance signal**
1. `list_guidance` — `end_timestamp` = decision date, `values_mode: "latest"`. The book as it stood, with no leakage.
2. `list_guidance_outcomes` — a later window, same filters. What those guides turned into.
3. `guidance_bias` — with `end_timestamp` at each decision date, so the score only knows prior reports.

**Leave it running**
`create_alert` on `list_guidance` (revisions) or `list_guidance_outcomes` (beats and misses), arguments minus timestamps — the alert owns the window. Keep `values_mode: "all"` so each delivery carries the prior print to compare against.

---

## Combine with

- **Wire / developments** — Why it matters. Wire wants the legal name (`Nvidia Corporation`), not `NVDA`.
- **`list_changes`** — How they wrote the release. Same ticker; match the document’s fiscal period from `source_metadata` (`list_guidance` at `full`).
- **Alerts** — `list_guidance` and `list_guidance_outcomes` only; `guidance_bias` is not alertable. See **Workflow**.

---

## Limitations

- Not every listed company is in the data. Empty is not “no outlook” and not “they always hit.”
- `latest` can still show an old child series next to a current print. Assemble the pack; don’t treat the first N rows as the slide.
- Empty `revenue` after a substring search ≠ no outlook. Read the pack.
- `source_types` does not filter `values[]`.
- Transcripts use an Ultralayer URL; press releases use the SEC URL.

---

## FAQ

**Why did `source_types: ["earnings_release"]` return a call?**  
The series once had a release. The newest row can still be the call. Read `values[].source_type`.

**Why is WMT `revenue` empty?**  
They may not guide a revenue-like line. Substring is the default (`revenue` also matches `revenue_growth`). Do not say they have no outlook. Pin `metric_key_exact_match: true` if you only want the level.

**Why doesn’t `resolved_count` match the outcomes list?**  
The list keeps the newest guide per line. Bias averages every guide that closed that night.

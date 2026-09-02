---
name: changes
description: Wording changes between one earnings press release and the last — what the company started saying, stopped saying, or now says differently. Operation is list_changes.
---

# Changes

`list_changes` compares a company’s latest earnings press release to the previous one. Each result is one thing they added, removed, or rewrote.

| Operation | Role |
|-----------|------|
| `list_changes` | Return matching wording changes |

Every filter works **without a ticker list**, so this is a market-wide screen as much as a single-filing read: who redefined a metric this quarter, who rewrote what the outlook assumes, who quietly dropped a risk. The material change is usually the one nobody headlined.

---

## When to use

**Use it for** the language, not the numbers: what this release says that last quarter’s did not, what they stopped mentioning, how a metric is now defined, how segments are cut, what the outlook assumes.

**Not this tool:** the outlook figure or who raised / lowered → `list_guidance`. Guide vs actual → `list_guidance_outcomes`. Headlines, rumor, or what the market is reacting to → Wire. Anything outside an earnings press release → `semantic_search` or your own web search.

---

## Use cases

**Realtime change tape** — What companies just started or stopped saying, market-wide, minutes after each release. No tickers.
```json
{
  "sort": "timestamp",
  "min_materiality_score": 0.5,
  "detail": "standard",
  "limit": 40
}
```

**Comparability breaks** — A redefined non-GAAP metric or a re-cut segment silently invalidates every model and screen built on the old basis, and it is stated once in the release. Highest-value screen here. INTU dropping share-based comp from non-GAAP scores 0.97; moving Mailchimp into its own segment scores 0.88.
```json
{
  "change_categories": ["metric_definition", "reporting_structure"],
  "sort": "materiality",
  "min_materiality_score": 0.7,
  "detail": "standard",
  "limit": 30
}
```

**What the outlook now assumes** — Not the figure, the basis: a deal folded in, a tariff line added, a multi-year target dropped. A guide that looks cut is sometimes just re-based. Then `list_guidance` for the number.
```json
{
  "change_categories": ["guidance_basis"],
  "sort": "materiality",
  "min_materiality_score": 0.5,
  "detail": "standard",
  "limit": 30
}
```

**Quiet de-risking** — What they stopped saying: a lawsuit no longer named, a KPI no longer highlighted, an assumption dropped. The proof is `prior_quote`.
```json
{
  "change_kinds": ["removed"],
  "sort": "materiality",
  "min_materiality_score": 0.5,
  "detail": "standard",
  "limit": 30
}
```

**Theme sweep** — No text search. One theme (tariffs, refunds, AI capex) lands across several categories; pass a few and read the summaries.
```json
{
  "change_categories": ["policy", "guidance_basis", "discrete_item", "results_attribution"],
  "sort": "materiality",
  "min_materiality_score": 0.5,
  "detail": "standard",
  "limit": 40
}
```

**Legal and one-off items** — Do not require `impact: ["negative"]`. Dropping a named lawsuit is often `legal` and `positive`. Big charges sometimes land in `discrete_item`, not `legal`.
```json
{
  "change_categories": ["legal", "discrete_item"],
  "sort": "materiality",
  "min_materiality_score": 0.7,
  "detail": "standard",
  "limit": 20
}
```

**One release, deep read** — Ticker + the fiscal year and quarter of that release. The prior period is the previous quarter (INTU FY26 Q4 vs Q3). Floor `0` when you want everything in one filing.
```json
{
  "symbols": ["INTU"],
  "fiscal_year": 2026,
  "fiscal_quarter": 4,
  "min_materiality_score": 0,
  "detail": "standard",
  "limit": 40
}
```

**Watchlist** — Up to 8 tickers. `materiality` puts the biggest changes first (including older ones); `timestamp` puts the newest first.
```json
{
  "symbols": ["NVDA", "INTU", "WMT", "DE"],
  "start_timestamp": "2026-06-01T00:00:00Z",
  "sort": "materiality",
  "min_materiality_score": 0.5,
  "detail": "standard",
  "limit": 30
}
```

---

## Response detail

Default is `full`. `detail` only drops fields. Price does not change.

| Level | What you get |
|-------|----------------|
| `essential` | Ticker, when it published, fiscal period, kind, category, summary. No quotes. |
| `standard` | That plus company name, both quotes, scores, impact, prior period, and the SEC URLs |
| `full` | That plus `cik` and extra source fields (`document_type`, fiscal period on the source) |

Use **`standard`** with an LLM — the quotes are already there. Use `essential` when you only need the one-line summary. Use `full` when a machine needs `cik` or the extra source fields.

At `standard`, `source_metadata` is null. The URL is still on the source.

---

## What you get back

Each item is one change. Default `limit` is 10 (max 100). Default `min_materiality_score` is **0.4**.

| Field | Notes |
|-------|-------|
| `symbol` / `company_name` / `cik` | `cik` only at `full` |
| `timestamp` | When we recorded the change. **Time filters use this.** |
| `source_timestamp` | When the company published the current release. Returned, not filtered. |
| `fiscal_year` / `fiscal_quarter` | Period the **current** release is about. Quarter is null on a full-year report. |
| `prior_fiscal_year` / `prior_fiscal_quarter` | The previous release. `fiscal_quarter=4` is Q4 vs Q3, not the full year. |
| `change_kind` | `added` / `removed` / `modified` |
| `change_category` | What kind of disclosure changed |
| `summary` | One sentence. Trust the quote if they disagree. |
| `current_quote` / `prior_quote` | The sentence in that release. Null on a removal / addition. |
| `materiality_score` | `0-.2` reword; `.2-.4` weak; `.4-.6` mod; `.6-.8` high; `.8-1` must-read |
| `impact` | Whether the wording change itself reads better or worse for the company — not a price call |
| `current_source` / `prior_source` | Always both, with a URL to each release |

`added` has `current_quote` only. `removed` has `prior_quote` only — read it. `modified` has both.

---

## Filters and sort

Filters combine with AND. Values inside a list are OR. Tickers are case-insensitive, max 8. Categories max 8. A category that is not in the list is rejected.

| Param | Notes |
|-------|-------|
| `start_timestamp` / `end_timestamp` | On `timestamp`. `end_timestamp: null` means now. A future end is treated as now. A future start is rejected. |
| `symbols` + `fiscal_year` + `fiscal_quarter` | One release. Year alone returns every pair in that year. |
| `change_kinds` | `added` / `removed` / `modified` |
| `change_categories` | See **Categories** |
| `impact` | `positive` / `negative` / `neutral`. Do not filter on this alone — the same kind of change can be positive at one company and negative at another. |
| `min_materiality_score` | Inclusive. Default **0.4** skips rewording and weak rows. Pass `0` to turn the floor off. |

| `sort` | Order |
|-------|--------|
| `timestamp` (default) | Newest first |
| `materiality` | Highest score first, then newest |

---

## Categories

Category is *what changed*, not whether it is good news. Taking a named lawsuit out of the risk list is often `legal` or `risk`, `removed`, and `positive`.

| Category | Use for |
|----------|---------|
| `metric_definition` | How a non-GAAP or adjusted metric is calculated |
| `disclosed_metric` | A named highlight they started, stopped, or renamed |
| `reporting_structure` | Segments or what is in the report |
| `guidance_basis` | Assumptions behind the outlook — not a raise or a cut |
| `policy` | Tariffs, tax rules, regulation |
| `discrete_item` | Impairments, charges, restructuring, one-offs |
| `legal` | Lawsuits and other legal actions |
| `capital_action` | Buybacks, financing, deals, spins |
| `risk` / `catalyst` | A threat vs an upcoming event. Good or bad does not pick the label. |

`accounting_policy`, `control_and_audit`, `concentration`, and `compensation` rarely return rows here. Empty on those is not “the company is clean.”

`product` and `catalyst` can flood a list that includes drug or chip names. Skip `business_description` and low-scoring `results_attribution` unless you asked for them.

---

## Workflow

Chains worth more than a single call.

**Earnings-night read on one name**
1. `list_changes` — ticker + that release’s fiscal year and quarter, floor `0`. Everything that moved in the language.
2. `list_guidance` — same ticker, for the figures the wording sits around.
3. `list_guidance_outcomes` — how the period that just closed scored against the guide they set a quarter ago.
4. Wire for how the tape read it; your own web search for the sell-side take.

**Comparability break → fix the model**
1. `list_changes` — `metric_definition` and `reporting_structure`, market-wide, floor 0.7.
2. Read `prior_quote` against `current_quote` for the old and new basis, and note the fiscal period the change takes effect.
3. `list_guidance` — whether the outlook is now stated on the new basis. A guide that looks cut can just be re-based.
4. Restate history before comparing anything across that boundary, and re-check any screen that ranks on the affected metric.

**Theme exposure across the market**
1. `list_changes` — theme categories, `sort: "materiality"`, no tickers. Who is talking about it and how it hit them.
2. Group the tickers that come back; the same theme appears under several categories.
3. `identify_stakeholders` — the full winner/loser map, beyond the filers who happened to word it into a release.
4. `list_guidance` — whether any of them put the theme into the outlook basis, which is where it becomes a number.

**Quiet de-risking sweep**
1. `list_changes` — `change_kinds: ["removed"]`, floor 0.5, no tickers.
2. Read `prior_quote` on each. Removal means they stopped saying it, not that the underlying item resolved.
3. Wire or your own web search — confirm whether the case settled, the KPI moved into a table, or the risk simply went unmentioned.

**Leave it running**
`create_alert` with `path: "/v0/filings/list_changes"`, same filters minus timestamps. `metric_definition` + `reporting_structure` at a high floor is a low-noise, high-value inbox; pin `detail` and `min_materiality_score`.

---

## Combine with

- **`list_guidance`** — The outlook and how it was revised. Same ticker; match fiscal year / quarter to the report period on the guidance source (`source_metadata`, which needs `list_guidance` at `full`).
- **Wire** — The headline.
- **`semantic_search`** — Only if you need a point-in-time or utility-filtered passage.
- **Alerts** — Alertable. Threshold counts changes; the alert owns the window. See **Workflow**. Wire remains the headline monitor.

---

## Limitations

- Not every listed company is in the data.
- An empty list can mean nothing matched, the floor is too high, or that release is not in yet. Do not treat it as “nothing changed.”
- Rows show up a few minutes after the release. Then nothing until the next earnings press release.
- No keyword search. Use categories and read the summaries.
- `removed` often means they stopped highlighting something, not that they stopped reporting it.
- One theme can appear under more than one category.

---

## FAQ

**Why did a 0.6 floor miss HD’s IEEPA guidance-basis change?**  
It is 0.56. Default 0.4 is safer if you care about outlook assumptions.

**I filtered to the press-release time and got nothing.**  
Time filters use `timestamp` (when we recorded it), not `source_timestamp` (when they published). INTU’s release is `20:02:29Z`; the rows show up at `20:04:20Z`.

**Does `removed` mean they stopped reporting the number?**  
No. They may have stopped highlighting it. The figure can still be in a table. Deere no longer *names* the FTC suit — that is not the same as the case ending.

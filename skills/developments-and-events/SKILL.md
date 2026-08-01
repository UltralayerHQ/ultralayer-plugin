---
name: developments-and-events
description: Structured market narratives — impact-scored, source-cited developments clustered into longer-lived events. Operations are search_developments, retrieve_development, search_events, and retrieve_event_developments.
---

# Developments and events

**Developments** are structured narrative units (name, description, scores, stakeholders with signed impact, sources with quotes) versioned over time. **Events** are longer-lived story containers that group related developments into a lifecycle (earnings cycle, M&A process, regulatory campaign, geopolitical conflict).

| Operation | Role |
|-----------|------|
| `search_developments` | Semantic or filter search over developments — returns packages |
| `retrieve_development` | Package for one known `development_id` (optional as-of) |
| `search_events` | Semantic or filter search over events; embeds recent development summaries |
| `retrieve_event_developments` | Event metadata + developments timeline |

---

## When to use

**Use this surface when you need to:**
- Explain a market move with who wins/loses (signed `impact_score` per ticker) and cited sources
- Track a storyline lifecycle via an `event_id`
- Scan for surprising / important developments
- Filter by public-company ticker involvement
- Backtest “what was known as of T” with `end_timestamp`

**Do not use when you need:**
- Realtime novelty-clustered headlines → Wire
- Quoteable document passages → `semantic_search`
- A single-entity dossier → `retrieve_entity`
- Exact entity-registry monitoring by `canonical_name` → Wire (developments use **tickers**, not Ultralayer entity ids)

---

## Use cases

**Ticker blast radius** — Find developments that hit a ticker hard in either direction, with signed impacts and quotes.
```json
{
  "stakeholder_symbol": "TSLA",
  "min_impact_score": 0.5,
  "max_impact_score": -0.5,
  "limit": 8
}
```

**Negative for one ticker** — Isolate clearly negative impact on a single name.
```json
{
  "stakeholder_symbol": "GOOGL",
  "max_impact_score": -0.5,
  "limit": 8
}
```

**Surprising geopolitics** — Scan for high-surprise security developments, not every geopolitics mention.
```json
{
  "development_type": "geopolitical_security",
  "min_surprise_score": 0.6,
  "start_timestamp": "2026-07-20T00:00:00Z",
  "limit": 10
}
```

**Theme → development evidence** — Ask in natural language and get structured impact packages back.
```json
{
  "query": "Strait of Hormuz shipping disruption oil",
  "start_timestamp": "2026-07-01T00:00:00Z",
  "limit": 5
}
```

**Theme → event clusters** — See how a theme groups into longer-lived stories, not just one-off updates.
```json
{
  "query": "Middle East oil",
  "limit": 5,
  "developments_per_event": 2
}
```

**M&A scan** — List active deal storylines without inventing your own type labels.
```json
{
  "event_type": "merger_acquisition",
  "limit": 5,
  "developments_per_event": 1
}
```

**Important prints** — Desk scan for high-importance developments without a theme query.
```json
{
  "min_importance_score": 0.8,
  "limit": 10
}
```

**Lifecycle expand** — Once you have an event id, pull the full timeline (scheduled → print → follow-ons).
```json
{
  "event_id": 125,
  "limit": 20,
  "include_future_developments": true
}
```

**As-of / backtest search** — Ask what developments were known as of a past timestamp.
```json
{
  "query": "Tesla Q2 earnings margins",
  "end_timestamp": "2026-07-22T21:00:00Z",
  "limit": 10
}
```

**As-of single development** — Reconstruct one development’s package as it stood at a prior time.
```json
{
  "development_id": 1793,
  "end_timestamp": "2026-07-22T21:00:00Z"
}
```

---

## Mental model

```text
Event (long-lived story)
  └── Development (versioned fact/update in that story)
        ├── scores: importance / surprise / confidence
        ├── occurrence_*: real-world time + status
        ├── stakeholders[]: PUBLIC_COMPANY + symbol + impact ∈ [-1,1]
        └── sources[]: citations with quotes
```

A development can belong to **multiple events**. Always read `events` / `event_ids`.

Three clocks (do not conflate):

| Field | Meaning |
|-------|---------|
| `timestamp` | System/version time — used for backtest `start`/`end_timestamp` and “latest version” collapse |
| `source_timestamp` | Latest source used for the development |
| `occurrence_timestamp` | Estimated real-world when it happened (or will) — lifecycle ordering and `include_future_developments` |

---

## Response detail

Most operations accept `detail`. **Default is `full`.** Reduced levels **drop keys** only — surviving fields keep the same names and paths as `full`. Pricing does not change with `detail`; this is about **response token cost** (development packages are heavy).

### search_developments / retrieve_development / retrieve_event_developments

Levels: `full` | `standard` | `essential`.

| Level | What you get | Rough token savings vs `full` |
|-------|----------------|-------------------------------|
| `essential` | Identity, timing, name/description, stakeholder impacts (no confidence/sources), thin event refs | ~75% (search/retrieve development); ~68% (`retrieve_event_developments`) |
| `standard` | essential + type/status, heuristic scores, citable sources (title/url; no quotes), fuller event summaries | ~39% (search/retrieve development); ~38% (`retrieve_event_developments`) |
| `full` | Every field including rationale and source quotes | — |

**Prefer `detail: "standard"`** for normal agent calls. Use `essential` for scanning many packages; use `full` when you need rationale or quoted evidence in the payload.

```json
{
  "stakeholder_symbol": "TSLA",
  "min_impact_score": 0.5,
  "max_impact_score": -0.5,
  "limit": 8,
  "detail": "standard"
}
```

### search_events

Levels: `full` | `essential` only (no `standard` — the middle tier barely saved tokens on this shape).

| Level | What you get | Rough token savings vs `full` |
|-------|----------------|-------------------------------|
| `essential` | Event id/name/description + recent development summaries (id / occurrence / name / description) | ~33% |
| `full` | Includes `event_type` and richer recent-development fields (scores, types, …) | — |

Prefer `detail: "essential"` when scanning events; use `full` when you need event type or summary scores.

---

## search_developments

Optional `query`: semantic over development descriptions, ranked by cosine (cosine not returned in the response). No query: latest version per `development_id` matching filters, ordered by `timestamp` DESC.

Returns development packages shaped by `detail` (default `full`; prefer `standard` — see **detail** above). You do not need `retrieve_development` after a search hit unless you need as-of reconstruction for a known id.

| Param | Notes |
|-------|-------|
| `query` | Optional. Max 2000 chars. |
| `stakeholder_symbol` | **Exact ticker** as stored (`TSLA`, `GOOGL`, `SGRO.L`). Max 8 chars. `tsla` → `[]`. Not Ultralayer canonical names. |
| `min_impact_score` / `max_impact_score` | Alone: threshold. **Together: directional OR** (`impact >= min OR impact <= max`) — requires `min >= max` (e.g. `0.5` + `-0.5`). |
| `min_importance_score` / `min_surprise_score` / `min_confidence_score` | Importance ~0.8+ is a strong desk filter |
| `development_type` | Exact free-form string (`financial_update`, `corporate_action`, `geopolitical_security`, …). Invented values silently return `[]`. Discover types from results. |
| `start_timestamp` / `end_timestamp` | On development `timestamp`. Future start → 422. Future end → realtime. |
| `start_occurrence_timestamp` / `end_occurrence_timestamp` | Real-world occurrence clock |
| `limit` | Default 5, max 20 |
| `detail` | Default `full`. Prefer `standard`. |

Packages are large at `full` (rationale + per-stakeholder quotes). Prefer tight `limit` (3–10) and `detail: "standard"`; expand event context with `retrieve_event_developments` when you need the storyline.

`occurrence_status` values seen: `occurred`, `scheduled`, `announced`, `ongoing`, `rumored` — no status filter param.

---

## retrieve_development

Required: `development_id`. Optional: `end_timestamp` for as-of (latest version with `timestamp <= end`). Optional: `detail` (default `full`; prefer `standard`).

Use when you already have an id (from an alert, prior run, or citation) or need as-of reconstruction. Search results already include the package at the requested `detail` — do not re-fetch the same id after `search_developments` unless you need a different `detail` or as-of.

Missing id → 404. As-of before the development existed → 404. Future `end_timestamp` → realtime.

---

## search_events

Optional `query`: semantic over event descriptions. No query: ordered by most recent development activity.

Timestamp filters gate on **development activity**: an older event still appears if it had a development recorded in-window. No activity → `[]`.

| Param | Notes |
|-------|-------|
| `query` | Optional. Max 2000 chars. |
| `event_type` | Exact free-form (`merger_acquisition`, `earnings`, `product_launch`, `executive_succession`, …) |
| `limit` | Default 5, max 10 |
| `developments_per_event` | Default 3, max 5, **0 allowed** (event cards only — cheap scanning) |
| `include_future_developments` | Default true. Filters on `occurrence_timestamp` vs now/end — does not drop past-dated `scheduled` items |
| `detail` | `full` \| `essential` only (no `standard`). Prefer `essential` when scanning. |

Embedded `recent_developments` are summaries (no rationale/stakeholders/sources). Escalate with `retrieve_event_developments` or `search_developments` for development packages.

---

## retrieve_event_developments

Required: `event_id`. Returns event metadata + developments ordered by `occurrence_timestamp` DESC. Packages follow `detail` (default `full`; prefer `standard`).

| Param | Notes |
|-------|-------|
| `limit` | Default 5, max 20 |
| `start_timestamp` / `end_timestamp` | Filter on development **recording** time (useful for polling new packages) |
| `include_future_developments` | Default true |
| `detail` | Default `full`. Prefer `standard`. |

Missing event → 404.

---

## Workflow

| User intent | Approach |
|-------------|----------|
| “Who is affected by X?” | `search_developments` → read stakeholder impacts; cite quotes |
| “What’s the Tesla earnings story this year?” | `search_developments` or `search_events` → take `event_id` → `retrieve_event_developments` |
| “Any big M&A?” | `search_events` with `event_type: merger_acquisition` |
| “Negative for GOOGL?” | `stakeholder_symbol: GOOGL`, `max_impact_score: -0.5` |
| “What just printed that’s important?” | No-query search + `min_importance_score: 0.8` |
| “Headline triage / first print” | Wire first; escalate here for structured impact |
| “Reconstruct this development as of T” | `retrieve_development` with `end_timestamp` |

1. Decide grain: atomic impact update (developments) vs storyline cluster (events).
2. Prefer ticker + score filters for desk questions; prefer semantic query for themes.
3. Keep `limit` small; set `detail: "standard"` (or `essential` on `search_events`); set `developments_per_event` low (or 0) when scanning events.
4. `search_developments` already returns packages at your chosen `detail` — do not call `retrieve_development` on those ids unless you need as-of or a richer `detail`. Expand storylines with `retrieve_event_developments`.
5. Collapse near-duplicate developments under the same event before briefing.
6. Hand off to Wire for live novelty, semantic search for raw evidence, `retrieve_entity` for identity packages.

---

## Limitations

- Near-duplicate developments can exist as separate ids for the same real-world fact — dedupe by event + occurrence window + name similarity.
- Stakeholders are **PUBLIC_COMPANY + ticker only** — no countries/commodities/people as stakeholders.
- Ticker strings are exact and case-sensitive (`TSLA` ≠ `tsla`), max 8 chars.
- `development_type` / `event_type` are not closed enums — inventing labels yields empty results.
- Cosine not exposed for thresholding.
- Packages are heavy at `full` — bad default for large scans without `detail: "standard"` / `essential`, tight limits, or `developments_per_event: 0`.
- `scheduled` status can linger after occurrence time.
- Not filterable by Ultralayer entity ids; use tickers or semantic query text.

---

## FAQ

**Q: Wire already has Tesla headlines — why developments?**  
A: Developments add multi-ticker impact scores, rationales, and event clustering. Wire tells you it printed; developments tell you who is in the blast radius and why.

**Q: Why did `development_type: "earnings"` return nothing?**  
A: Earnings prints are often typed `financial_update`; `earnings` appears more as an event_type. Discover types from hits.

**Q: `include_future_developments: false` still showed “scheduled” items.**  
A: Filter is on `occurrence_timestamp` vs now, not on status label. Past-dated schedules remain.

**Q: Can I filter by Ultralayer entity name `Tesla, Inc.`?**  
A: No. Use `stakeholder_symbol: "TSLA"` or semantic query text.

**Q: search_events with a July window returned an older long-running event — bug?**  
A: Timestamps gate development activity. Old events with new developments in-window correctly appear.

**Q: retrieve_development as-of before the print 404s.**  
A: Expected — no version existed yet.

**Q: search_events vs search_developments?**  
A: Events for clusters/lifecycles (summaries); developments for atomic updates + stakeholder math (full packages). Typical flow: search either → expand with `retrieve_event_developments` when you need the timeline; use `retrieve_development` only for a known id or as-of.

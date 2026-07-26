---
name: wire
description: Realtime and backtestable financial news wire — novelty-clustered headlines with entity mentions, sector and asset tags, and trust/importance scores. Operations are list_wire and wire_storyline.
---

# Wire

The Wire is Ultralayer’s structured news firehose: short, fact-dense headlines with novelty links, entity mentions, sector/asset tagging, and trust/importance scores. Public surface exposes **metadata + headline only** (no article body).

| Operation | Role |
|-----------|------|
| `list_wire` | Filter + return the most recent matching wire items |
| `wire_storyline` | Expand one item into its novelty ancestor chain + first-degree branches |

---

## When to use

**Use the Wire when you need to:**
- Scan what is moving now (earnings, guidance, geopolitics, rates, M&A)
- Monitor a company, commodity, country, sector, or publisher
- Deduplicate the firehose via `novelty` (`new` / `update` vs `duplicate`)
- Reconstruct how a story evolved (updates, corrections, cross-publisher copies)
- Backtest a window with `start_timestamp` / `end_timestamp`
- Build alert-ready conditions (same filter fields power `list_wire` alerts)

**Do not use the Wire when you need:**
- Passage-level evidence from article text → `semantic_search`
- A full entity package → `retrieve_entity`
- Semantic “find similar stories by meaning” — `list_wire` is filter/recency, not embedding search

---

## Use cases

**Realtime desk scan** — See the latest material earnings and guidance prints without the syndication noise.
```json
{
  "max_records": 25,
  "novelty": ["new", "update", "correction"],
  "categories": ["earnings", "guidance"],
  "min_importance_score": 0.5,
  "min_trustworthiness_score": 0.7
}
```

**Company name watch** — Pull only headlines where a specific company is a primary or major subject.
```json
{
  "max_records": 20,
  "entities": ["Tesla, Inc."],
  "entity_relevance": ["primary", "significant"],
  "novelty": ["new", "update", "correction"]
}
```

**Strong ± sentiment on a name** — Surface mentions that are clearly bullish or bearish on the company, not mild noise.
```json
{
  "entities": ["easyJet plc"],
  "entity_relevance": ["primary", "significant"],
  "min_entity_sentiment": 0.3,
  "max_entity_sentiment": -0.3,
  "max_records": 20
}
```

**Geopolitics → commodities** — Catch geopolitics that matter for energy and commodities.
```json
{
  "max_records": 20,
  "sectors": ["energy"],
  "asset_classes": ["commodities"],
  "categories": ["geopolitics"],
  "novelty": ["new", "update"],
  "min_importance_score": 0.5
}
```

**Issuer originals only** — Read what the company itself released, not second-hand rewrites.
```json
{
  "content_type": ["press_release"],
  "source_relationship": ["first_party"],
  "novelty": ["new", "update", "correction"],
  "max_records": 20
}
```

**Scheduled / future events** — Find upcoming earnings and policy dates, not just what already printed.
```json
{
  "recency": ["report_of_future"],
  "categories": ["earnings", "monetary"],
  "max_records": 20
}
```

**Backtest a window** — Reconstruct what looked important during a specific trading session.
```json
{
  "start_timestamp": "2026-07-21T14:00:00Z",
  "end_timestamp": "2026-07-21T18:00:00Z",
  "novelty": ["new", "update"],
  "min_importance_score": 0.6,
  "max_records": 50
}
```

**Storyline with syndication** — See how a story evolved and which outlets copied it.
```json
{
  "wire_id": 67337,
  "novelty": ["new", "update", "duplicate", "correction"]
}
```

---

## What you get back

Each item is a compact decision object. Headlines are usually one dense sentence with numbers, entities, and the delta.

| Field | Notes |
|-------|-------|
| `headline` | Primary content |
| `wire_id` | Pass to `wire_storyline` |
| `timestamp` | Wire clock — **filters** use this |
| `source_timestamp` | Publisher time — **list_wire sorts by this descending** |
| `novelty` + `prior_wire_id` | Cluster membership. `new` usually has `prior_wire_id: null` |
| `importance_score` / `trustworthiness_score` | 0–1 |
| `factuality` | `confirmed` / `reported` / `rumor` / `speculation` / `forecast` |
| `content_type` | `news`, `analysis`, `opinion`, `press_release`, `data_release` |
| `recency` | `realtime` / `report_of_past` / `report_of_future` |
| `source_relationship` | `first_party` / `reporting` / `second_hand` |
| `about` | `sectors`, `industries`, `countries` (ISO alpha-2 lowercase), `asset_classes` |
| `entities[]` | `canonical_name`, `entity_type`, `relevance`, `sentiment` (−1..1), `identifiers` |
| `publisher` | Exact string for filters; values are inconsistent (`bloomberg`, `seeking alpha`, `dow_jones`) |

Prefer `novelty: ["new","update","correction"]` and `max_records` 10–30 with tight filters.

---

## list_wire

Returns up to `max_records` (1–100, default 20) matching items, newest by `source_timestamp`.

### Filtration vs sort

- **Filters** apply on `timestamp` (wire clock).
- **Response order** is `source_timestamp` descending.
- A tight time window can include items whose source time looks slightly outside the window. Prefer slightly wider windows when backtesting.

Future `end_timestamp` values are treated as realtime (no upper bound).

### Novelty

| Value | Meaning | Typical `prior_wire_id` |
|-------|---------|-------------------------|
| `new` | First material print of a cluster | `null` |
| `update` | Material change / next beat | prior in chain |
| `duplicate` | Same substance, often another publisher | prior (often a `new`) |
| `correction` | Number/fact fix; rare but market-critical | prior wrong print |

Default monitoring set: `["new", "update", "correction"]` — cuts roughly half the firehose noise from cross-publisher copies while keeping beats and fixes.

Example: Lundbeck FDA Fast Track — `pr_newswire` item `novelty=new`, Reuters copy minutes later `novelty=duplicate` with `prior_wire_id` → the PR.

### Entity filter

`entities` is include-any.

1. Match is **case-insensitive exact `canonical_name`**, not tickers, not aliases.
2. Unknown names → **HTTP 422** with `unresolved_entities` + trigram `suggestions`. `TSLA` fails; suggestions may include `Tesla` — that is not necessarily the right company identity.
3. Short names can resolve to the **wrong** registry row and return **empty `[]` with no error**. `"Tesla"` / `"Apple"` → `[]`; `"Tesla, Inc."` / `"Apple Inc."` → real hits.
4. Copy `canonical_name` from a prior wire hit or `retrieve_entity`. Prefer `"easyJet plc"`, `"Nvidia Corporation"` when unsure.
5. Pair with `entity_relevance` and `min_entity_sentiment` / `max_entity_sentiment`. When both sentiment bounds are set, `min` must be **greater than** `max` (directional OR: strong positive **or** strong negative).

Sentiment bounds apply to the matching mention, not the whole item.

### Other high-value filters

| Filter | Notes |
|--------|-------|
| `categories` | Free-form. Common: `earnings`, `guidance`, `geopolitics`, `trade`, `m&a`, `regulation`, `macro`, `monetary`, `product`, `tech`, `litigation`. Wrong tags silently return fewer hits. |
| `sectors` | Closed GICS-like enum (`information_technology`, `energy`, …) |
| `industries` | Free-form as stored (e.g. `"semiconductors"`) |
| `countries` | Lowercase ISO-3166-1 alpha-2 |
| `asset_classes` | `equities`, `fixed_income`, `commodities`, `currencies`, `crypto`, `real_estate` |
| `publisher` | Exact match; discover from results |
| `content_type` + `source_relationship` | `press_release` + `first_party` for issuer originals |
| `factuality` | `rumor` / `speculation` for soft takes |
| `recency` | `report_of_future` for scheduled/future prints |
| `min_importance_score` | Unfiltered stream is full of 0.2–0.3 items |
| `min_trustworthiness_score` | Filters thin social sources |

Filters combine with AND across dimensions; within multi-value arrays, include-any (OR).

Keep filter lists tight: across `categories`, `entities`, `sectors`, `industries`, `countries`, `asset_classes`, and `publisher`, at most **12 values total** are allowed in one request.

---

## wire_storyline

Given a `wire_id`, returns the novelty storyline: linear `prior_wire_id` ancestors plus first-degree children of every chain node.

```json
{
  "seed_wire_id": 67583,
  "root_wire_id": 49200,
  "items": [ ]
}
```

Items are chronological by `source_timestamp` ascending.

| Param | Notes |
|-------|-------|
| `max_ancestor_depth` | Default 32, max 64. Caps the linear prior walk (including the seed). |
| `max_children` | Default 32, max 64. Caps first-degree children across the whole chain. |

### Topology

1. Walk seed → `prior_wire_id` → … until root, time-window break, or `max_ancestor_depth`.
2. Ancestors are walked even through duplicates so the chain stays connected; novelty filtering is applied when projecting the response (seed always kept).
3. Children: only direct children of chain nodes, capped by `max_children`.
4. `root_wire_id` is set only if the walk reached a row with `prior_wire_id: null`. **`root_wire_id: null` means truncated** (time window or depth).
5. Seed outside the requested time window → **404**. Unknown `wire_id` → **404**.

### Default novelty trap

`wire_storyline` defaults to `novelty: ["new", "update", "correction"]` — **duplicates excluded**. That hides cross-publisher fanout.

- Default: good for “how did the story evolve?”
- Include `"duplicate"` when you need “who else printed this?”

Example — easyJet earnings: default novelty → only the Reuters `new` seed; with duplicates → Reuters + Bloomberg + FT copies.

Example — earnings correction chain: `new` → `correction` → `update` → later conflicting `correction`. Storyline is how you catch conflicting prints.

Example — multi-day geopolitics: seed linked through consecutive updates to a root days earlier; a tight time window truncates the chain and returns `root_wire_id: null`.

---

## Workflow

| User intent | Approach |
|-------------|----------|
| “What’s moving markets right now?” | `list_wire` with novelty without duplicates, `min_importance_score` ≥ 0.5 |
| “Any news on Company X?” | Resolve canonical name → entities + primary/significant → storyline on the best hit |
| “Did guidance change?” | `categories: ["guidance","earnings"]`, then storyline |
| “Is this a correction?” | `novelty: ["correction"]` or storyline and look for correction nodes |
| “Who reported it first?” | Storyline with duplicates; earliest `new` / first_party press release is usually the root |
| “Oil / Red Sea / rates?” | `asset_classes` + `sectors` + geopolitics/macro categories |
| “What looked important on date D?” | Timestamp window + importance floor + novelty new/update |

1. Clarify scope (entity / sector / theme / time).
2. `list_wire` with novelty `new|update|correction`, an importance floor, and the tightest true filters.
3. If entity filter returns empty without 422, retry with a fuller canonical name.
4. Pick the highest-importance hit → `wire_storyline` (add `duplicate` if coverage breadth matters).
5. Brief from headlines + novelty path; escalate to `semantic_search` or the URL when body text is required.

---

## Limitations

- No full article text on the public wire surface.
- Entity filter is canonical-name exact; tickers/aliases fail or mis-resolve.
- Duplicate short names in the registry can silently return empty results.
- `publisher`, `industries`, and `categories` are messy real-world strings.
- Storyline is shallow by design (linear ancestors + one hop of children).
- Unfiltered `list_wire` is noisy — bias novelty + importance.
- `importance_score` is relative rank, not a guarantee.

---

## FAQ

**Q: `entities: ["Tesla"]` returned `[]` but Tesla news exists.**  
A: Short name likely resolved to a different identity. Use `Tesla, Inc.` from a prior hit’s `entities[].canonical_name`. Empty array ≠ error; 422 only on unresolved names.

**Q: Why did `TSLA` 422?**  
A: Tickers are not resolved. Use suggestions only as hints, then confirm the canonical company name.

**Q: Why doesn’t storyline show Bloomberg/FT copies?**  
A: Default novelty drops `duplicate`. Add it explicitly.

**Q: `root_wire_id` is null but items exist.**  
A: Ancestry was truncated by time window or `max_ancestor_depth`. Widen `start_timestamp`, raise the depth cap, or treat as a partial chain.

**Q: Sort order looks wrong vs IDs.**  
A: `list_wire` sorts by `source_timestamp` desc, not `wire_id`. Filtration still uses `timestamp`.

**Q: list_wire vs alerts?**  
A: Same filter vocabulary. The Wire is the pull API; alerts can poll `list_wire` conditions on a schedule.

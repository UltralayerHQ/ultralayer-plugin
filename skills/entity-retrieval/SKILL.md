---
name: entity-retrieval
description: Realtime context package for one entity — identity, recent high-relevance headlines, 14-day sentiment, and top co-mentions. Operation is retrieve_entity.
---

# Entity Retrieval

`retrieve_entity` returns a realtime context package for one Ultralayer entity. Single required param: `entity`. No time window, no `top_k` — the package shape is fixed.

| Operation | Role |
|-----------|------|
| `retrieve_entity` | Realtime dossier for one entity |

---

## When to use

**Use `retrieve_entity` when you need to:**
- Bootstrap a company / person / commodity / country / institution briefing
- Read 14-day mention sentiment without hand-rolling Wire aggregation
- Discover who co-moves in the news with the target

**Do not use when you need:**
- A filtered multi-entity news firehose → Wire
- PIT or utility-filtered corpus passages → `semantic_search`
- Point-in-time / as-of packages → not supported (realtime only)
- Sparse / quiet entities → fails with `insufficient_entity_activity`

---

## Use cases

**Company dossier seed** — One call for recent headlines, 14-day sentiment, and who else is in the story.
```json
{ "entity": "Tesla, Inc." }
```

**Person → affiliated universe** — See which companies and projects dominate the news around a person.
```json
{ "entity": "Elon Musk" }
```
Follow co-mentions (`Tesla, Inc.`, `SpaceX`, `xAI`) with more `retrieve_entity` or Wire.

**Commodity / macro driver** — Brief a commodity and discover the geopolitics and related markets moving with it.
```json
{ "entity": "Brent Crude" }
```
Use co-mentions (Iran, Hormuz, WTI) to choose Wire entity filters and semantic queries.

**Institution** — Quick context pack for a policy body that drives markets.
```json
{ "entity": "Federal Reserve" }
```

**Country** — High-volume national tape: recent prints, sentiment trend, and co-moving names.
```json
{ "entity": "Iran" }
```

---

## What you get back

| Section | Notes |
|---------|-------|
| `entity` | `entity_id`, `canonical_name`, `entity_type` |
| `recent_wire_items` | Up to 5 teasers; novelty `new\|update\|correction`; entity is `primary` or `significant`; includes `wire_id` for storyline |
| `sentiment_chart` | Exactly 14 UTC calendar-day bars, oldest → newest. Relevance-weighted mean ∈ [-1, 1], weighted std, `mention_count`. Empty days stay with `weighted_sentiment: null`. Current day is a partial bar. |
| `co_mentions` | Top peers with `share` (fraction of recent high-relevance wire that also mentions them at primary/significant) |

Requires sufficient recent wire activity; sparse entities return 422 `insufficient_entity_activity`.

---

## How to read each section

**Recent wire items.** Decision-grade headlines. Duplicates stripped. Use `wire_id` → `wire_storyline` for story evolution.

**Sentiment chart.** `end_timestamp` is the exclusive end of the UTC day (bar for 2026-07-23 ends at `2026-07-24T00:00:00Z`). Interpret with `mention_count` and `weighted_std` — one-mention days and high std are noisy. Mega-entities (countries) show flatter averages.

**Co-mentions.** “Who shares the recent high-relevance tape?” Near-duplicate registry rows often both appear (`SpaceX` + `Space Exploration Technologies Corp.`, `United States` + `United States of America`). Collapse when briefing.

Works well for `COMPANY`, `PERSON`, `COMMODITY`, `COUNTRY`, `GOVERNMENT`. Products (e.g. `"ChatGPT"`) often fail the activity gate.

---

## Workflow

| User intent | Approach |
|-------------|----------|
| “Brief me on X” | Package → recent wires + sentiment turn + co-mentions |
| “Who is moving with oil / AI / this CEO?” | Read `co_mentions`; optionally package those peers |
| “Is tape tone improving?” | Walk `sentiment_chart` with mention_count/std |
| “Watch every mention going forward” | Use Wire (and alerts) |

1. Call `retrieve_entity`. Brief from: recent wires → sentiment turn → co-mentions.
2. Hand off: Wire for monitoring/storylines; your native web search tool for ordinary evidence; `semantic_search` only for PIT or utility-filtered corpus hits.

---

## Limitations

- Realtime only — no as-of.
- One entity per call.
- Activity gate hides quiet names.
- Public input is name only (passing `entity_id` as a string does not work).
- Recent strip is fixed at 5 non-duplicate high-relevance items.
- Co-mentions surface unmerged duplicate identities.
- Not a substitute for Wire novelty clustering or semantic passage retrieval.

---

## FAQ

**Q: Why do SpaceX and “Space Exploration Technologies Corp.” both show as co-mentions?**  
A: Incomplete entity merges. Treat as one org in the narrative unless you need registry fidelity.

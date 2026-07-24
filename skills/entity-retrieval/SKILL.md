---
name: entity-retrieval
description: Realtime context package for one entity — identity, recent high-relevance headlines, 14-day sentiment, top co-mentions, and structured attributes. Operation is retrieve_entity.
---

# Entity Retrieval

`retrieve_entity` returns a realtime context package for one Ultralayer entity. Single required param: `entity` (case-insensitive **canonical name**). No time window, no `top_k` — the package shape is fixed.

| Operation | Role |
|-----------|------|
| `retrieve_entity` | Realtime dossier for one entity |

---

## When to use

**Use `retrieve_entity` when you need to:**
- Bootstrap a company / person / commodity / country / institution briefing
- Resolve the correct `canonical_name` + `entity_id` before Wire filters
- Read 14-day mention sentiment without hand-rolling Wire aggregation
- Discover who co-moves in the news with the target
- Pull attribute bags (CEO, ticker, products, market cap, …) as starting facts — then verify

**Do not use when you need:**
- A filtered multi-entity news firehose → Wire
- Quoteable document passages → `semantic_search`
- Point-in-time / as-of packages → not supported (realtime only)
- Sparse / quiet entities → fails with `insufficient_entity_activity`

---

## Use cases

**Company dossier seed** — One call for recent headlines, 14-day sentiment, who else is in the story, and structured attributes.
```json
{ "entity": "Tesla, Inc." }
```

**Resolve the name before Wire** — Get the exact registry name so later company watches don’t silently miss.
```json
{ "entity": "Nvidia Corporation" }
```
Then reuse `entity.canonical_name` exactly in `list_wire` entity filters.

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
| `entity` | `entity_id`, `canonical_name`, `entity_type` — **copy `canonical_name` into Wire / later calls** |
| `recent_wire_items` | Up to 5 teasers; novelty `new\|update\|correction`; entity is `primary` or `significant`; includes `wire_id` for storyline |
| `sentiment_chart` | Exactly 14 UTC calendar-day bars, oldest → newest. Relevance-weighted mean ∈ [-1, 1], weighted std, `mention_count`. Empty days stay with `weighted_sentiment: null`. Current day is a partial bar. |
| `co_mentions` | Top peers with `share` (fraction of recent high-relevance wire that also mentions them at primary/significant) |
| `attributes` | NER-aggregated facts (CEO, TICKER, PRODUCTS, MARKET_CAP, …). Keys vary; can be `{}` |

Requires sufficient recent wire activity; sparse entities return 422 `insufficient_entity_activity`.

---

## Name resolution

Match is **case-insensitive exact `canonical_name`**. Tickers and loose aliases are not resolved.

| Input | Result |
|-------|--------|
| `"Tesla, Inc."` / `"tesla, inc."` | Success → root `Tesla, Inc.` |
| `"Tesla"` / `"Apple"` / `"NVIDIA"` | Often resolves to a different sparse registry row → **422 `insufficient_entity_activity`** |
| `"TSLA"` / numeric id as string | **422 `unresolved_entities`** + suggestions (can be noisy) |
| `"Nividia Corporation"` (typo) | 422 + useful suggestion `"Nvidia Corporation"` |
| `"OpenAI"`, `"SpaceX"`, `"Gold"`, `"Elon Musk"` | Success when that string **is** the canonical |

Short brand names that look right can fail the activity gate because they hit the wrong identity. Prefer full legal-style names for listed cos (`"Apple Inc."`, `"Nvidia Corporation"`, `"easyJet plc"`). Best practice: take `canonical_name` from a prior Wire hit or a successful package and reuse it.

Unknown names → `unresolved_entities` + `suggestions` (candidates to retry, not confirmed identities). Sparse-but-resolved names → `insufficient_entity_activity` with no suggestions — often means “wrong short name,” not “entity is quiet.”

---

## How to read each section

**Recent wire items.** Decision-grade headlines. Duplicates stripped. Use `wire_id` → `wire_storyline` for story evolution.

**Sentiment chart.** `end_timestamp` is the exclusive end of the UTC day (bar for 2026-07-23 ends at `2026-07-24T00:00:00Z`). Interpret with `mention_count` and `weighted_std` — one-mention days and high std are noisy. Mega-entities (countries) show flatter averages.

**Co-mentions.** “Who shares the recent high-relevance tape?” Near-duplicate registry rows often both appear (`SpaceX` + `Space Exploration Technologies Corp.`, `United States` + `United States of America`). Collapse when briefing.

**Attributes.** Useful starting facts, not a fundamentals database. Companies are densest; commodities/countries often thin or empty. Lists are synonym-heavy. Can flip quickly off NER (e.g. CEO field updating with succession headlines) — treat as hypothesis to verify.

Works well for `COMPANY`, `PERSON`, `COMMODITY`, `COUNTRY`, `GOVERNMENT`. Products (e.g. `"ChatGPT"`) often fail the activity gate.

---

## Workflow

| User intent | Approach |
|-------------|----------|
| “Brief me on X” | Package → recent wires + sentiment turn + co-mentions + verified attributes |
| “What’s the right entity name for Wire?” | Call here; reuse `canonical_name` exactly |
| “Who is moving with oil / AI / this CEO?” | Read `co_mentions`; optionally package those peers |
| “Is tape tone improving?” | Walk `sentiment_chart` with mention_count/std |
| “Give me fundamentals” | Attributes are a seed only — confirm via semantic search / trusted wire |
| “Watch every mention going forward” | After resolving the name, use Wire (and alerts) |

1. Choose the best canonical guess (full legal-style name for listed companies).
2. Call `retrieve_entity`. On `insufficient_entity_activity`, retry fuller forms before concluding the entity is quiet. On `unresolved_entities`, retry the best suggestion.
3. Brief from: recent wires → sentiment turn → co-mentions → attributes (facts to verify).
4. Hand off: Wire for monitoring/storylines; semantic search for evidence passages.

---

## Limitations

- Realtime only — no as-of.
- One entity per call.
- Activity gate hides quiet names and punishes ambiguous short names.
- Public input is name only (passing `entity_id` as a string does not work).
- Recent strip is fixed at 5 non-duplicate high-relevance items.
- Attributes are NER aggregates: incomplete, contradictory, occasionally wrong.
- Co-mentions surface unmerged duplicate identities.
- Not a substitute for Wire novelty clustering or semantic passage retrieval.

---

## FAQ

**Q: Why does `"Tesla"` fail with `insufficient_entity_activity` but `"Tesla, Inc."` works?**  
A: Short `"Tesla"` matches a different sparse registry entity. Same trap as Wire’s empty `[]` on short names — here it surfaces as an activity 422.

**Q: Can I pass `TSLA` or the numeric `entity_id`?**  
A: No. Tickers and id-strings → `unresolved_entities`. Use the canonical name.

**Q: Why is Gold’s attribute `COMPANY: Robinhood Markets, Inc.`?**  
A: Attribute noise from document NER. Cross-check before citing.

**Q: Why do SpaceX and “Space Exploration Technologies Corp.” both show as co-mentions?**  
A: Incomplete entity merges. Treat as one org in the narrative unless you need registry fidelity.

**Q: Empty `attributes` on Brent / United States — broken?**  
A: Common for non-company types. Lean on wires + co-mentions instead.

**Q: Are attributes filings-grade facts?**  
A: No. Verify material facts before citing.

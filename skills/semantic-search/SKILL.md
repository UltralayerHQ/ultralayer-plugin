---
name: semantic-search
description: Point-in-time and utility-filtered passage retrieval over Ultralayer’s financial corpus. Operation is semantic_search (1–8 queries). Not a replacement for your native web search tool.
---

# Semantic search

`semantic_search` returns **quoteable passages** from Ultralayer’s indexed financial corpus (news, transcripts, social, and related sources). Each result is one **document** with ordered **snippets**, scores, and source provenance.

It is **not** a general research tool. Your native web search tool (Perplexity, OpenAI, Bing, and similar) wins on coverage and depth for ordinary research. Use Ultralayer semantic search only when you need **point-in-time safety** or **utility filtration** on this corpus.

| Operation | Role |
|-----------|------|
| `semantic_search` | 1–8 queries → passages packed under one global `max_tokens` budget, with optional PIT window and utility / other filters |

---

## When to use

**Use `semantic_search` only when you need one of:**

1. **Point-in-time (PIT) research** — Reconstruct what was knowable as of a cutoff without later leakage. Filters run on system knowledge time (`timestamp`) at **second-level** granularity. Responses also carry publication time (`source_timestamp`). Many other “historical” or backtest APIs are coarser (often day-level) or filter only on published time; Ultralayer’s PIT window is second-level on **ingestion / knowable time**, with published time available alongside.
2. **Utility filtration** — A broad theme where you want only high-information passages (dense guidance, numbers, substance), not boilerplate or promotional copy. Raise `min_utility_score` (e.g. 0.7).

**Do not use `semantic_search` when:**

- You need broad coverage, depth, filings, prices, or open-web evidence → **your native web search tool**
- You need what just broke / novelty clusters → Wire
- You need exact company-name monitoring → Wire entities or `retrieve_entity`
- You need a guaranteed entity match — match is by meaning, not registry id

If the question is not PIT-required and does not need utility filtration, **prefer your native web search tool**.

---

## Use cases

**PIT — market consensus before an event** — Reconstruct what the market believed *before* something materialized (no later leakage). `end_timestamp` is the cutoff: only what was knowable at that instant.
```json
{
  "queries": ["Nvidia data center GPU demand outlook"],
  "max_tokens": 3000,
  "start_timestamp": "2026-07-01T00:00:00Z",
  "end_timestamp": "2026-08-15T20:00:00Z",
  "detail": "standard"
}
```

**Utility filtration on a general query** — The query is deliberately broad. Raise `min_utility_score` so you keep high-substance passages and drop promotional / boilerplate hits that a general query otherwise surfaces.
```json
{
  "queries": ["artificial intelligence"],
  "max_tokens": 3000,
  "min_utility_score": 0.8,
  "detail": "standard"
}
```

**Multi-facet PIT research** — Several angles in one call under one budget (still only when PIT or utility is the reason).
```json
{
  "queries": [
    "Tesla Q2 automotive margins",
    "Tesla Robotaxi Cybercab rollout",
    "Tesla Optimus humanoid robot guidance"
  ],
  "max_tokens": 5000,
  "start_timestamp": "2026-07-01T00:00:00Z",
  "end_timestamp": "2026-07-15T00:00:00Z",
  "detail": "standard"
}
```

---

## What you get back

Each hit is a **document**. Matching passages are in `snippets`.

| Field | Notes |
|-------|-------|
| `snippets` | Ordered passage texts — primary value |
| `snippet_cosine_similarities` | Parallel to `snippets` (full detail only). Document rank = `max(...)` |
| `o200k_base_n_tokens` | Token count of returned snippets (full detail only) |
| `utility_score` | 0–1. Default filter floors at **0.4** |
| `sentiment_score` | −1..1 document-level tone (not Wire entity-mention sentiment) |
| `canonical_entity_names` | Can be empty even when the text discusses a company |
| `document_id` | One result per document |
| `timestamp` | System knowledge / ingestion time — **this is what time filters use** |
| `source` | Includes `source_timestamp` (published time), `url`, `source_table`, `source_metadata` |

Prefer specific queries + modest `max_tokens` (2000–5000 for one query). Quote from `snippets`.

---

## Response detail

`semantic_search` accepts `detail`. **Default is `full`.** Reduced levels **drop keys** only — surviving fields keep the same names and paths as `full`. Pricing does not change with `detail`; this is about **response token cost** in the agent context.

| Level | What you get | Rough token savings vs `full` |
|-------|----------------|-------------------------------|
| `essential` | `title` + `snippets` + source timestamp | highest savings |
| `standard` | essential + citable source (`source_table`, `source_id`, `url`, and `source_metadata.title` when present) | medium |
| `full` | Every field (scores, entity names, document id, snippet similarities, token count, full source metadata, …) | — |

**Prefer `detail: "standard"`** for normal agent calls. Use `essential` when stuffing many hits into context; use `full` only when you need scores, entity names, or full source metadata.

---

## Defaults that shape every call

| Param | Default | Implication |
|-------|---------|-------------|
| `detail` | `full` | Prefer `standard`. |
| `start_timestamp` | `2026-01-01T00:00:00Z` | Not “all history.” Set explicitly for recent or PIT windows. |
| `min_utility_score` | `0.4` | Raise (e.g. 0.7) when utility filtration is the reason for the call. |
| `max_tokens` | `2000`, ge 1000, le 50000 | Require ≥500 tokens per query. One call supports 1–8 queries under one budget. |

Future `end_timestamp` is treated as realtime (no upper bound). Past `end_timestamp` = as-of / PIT mode.

Returned documents are ordered by each document’s best snippet similarity.

---

## How to write queries

1. **Specific topical phrases beat bare names.** Prefer keyword-dense packs (`Tesla Q2 earnings margin guidance Robotaxi`) over a bare ticker.
2. Questions are fine when they encode entities + mechanism; fluff words rarely help.
3. **Facet when you have ≥2 distinct angles** that still need PIT or utility on this corpus.
4. Not an entity resolver. A query mentioning easyJet can still return related airline passages — meaning neighborhood, not registry match.

---

## Filters

| Filter | Notes |
|--------|-------|
| Time | Inclusive on snippet **`timestamp`** (knowledge / ingestion time), second-level. Default lower bound is 2026-01-01. Published time is on the source, not the filter clock. |
| `min_utility_score` | Default 0.4. Raise for denser earnings/guidance language. Not Wire `importance_score`. |
| `min_cosine_similarity` | Optional similarity floor. Aggressive floors can return `[]`. |
| `min_sentiment_score` **or** `max_sentiment_score` | Mutually exclusive. Document-level tone. |
| `entity_types` | Coarse overlap (`COMPANY`, `COMMODITY`, …). Does not pin a named company. |
| `source_metadata_filters` | Publisher only. Closed **exact** domain strings (`seekingalpha.com`, `www.cnbc.com`). Not Wire’s free-form publisher names. |

---

## Workflow

| User intent | Approach |
|-------------|----------|
| Ordinary research / quotes / filings / prices | **Your native web search tool**, not `semantic_search` |
| “What was knowable as of T?” | `semantic_search` with `end_timestamp` (and usually `start_timestamp`) |
| “Only high-substance hits on this theme” | `semantic_search` + raised `min_utility_score` |
| “Realtime headline triage” | Wire first |
| “Who printed first / novelty” | Wire |

1. Ask: do I need **PIT** or **utility filtration**? If neither, use your native web search tool (or Wire for live headlines).
2. If yes: set the time window explicitly; raise `min_utility_score` when that is the reason.
3. Prefer `detail: "standard"`. Quote from `snippets` with URL from `source.url`.

---

## Limitations

- Focused financial corpus — not the coverage or depth of your native web search tool.
- Passage retrieval, not novelty-aware news clustering.
- Not an exact entity filter; related peers/themes can appear.
- `canonical_entity_names` can be incomplete vs the text.
- Publisher filter is a closed enum of domain-style strings — different from Wire publishers.
- Default `start_timestamp` and `min_utility_score` silently shape every naive call.

---

## FAQ

**Q: Should I use this instead of my native web search tool?**  
A: Use this only when you need point-in-time retrieval or utility filtration. Otherwise use your native web search tool.

**Q: How is PIT different from “historical search” elsewhere?**  
A: Filters use second-level **system knowledge / ingestion time** (`timestamp`), so as-of queries do not leak later knowledge. Published time (`source_timestamp`) is returned for citation. Many other tools only offer coarser windows or published-time filters.

**Q: Why are Fed-rate results from months ago when I wanted “now”?**  
A: Default `start_timestamp` is 2026-01-01; ranking is similarity, not recency. Set a recent window.

**Q: Why did a high `min_cosine_similarity` return nothing?**  
A: Lower the floor or improve the query.

**Q: I searched easyJet and got Ryanair — bug?**  
A: Meaning neighborhood for airline + fuel. For name-locked monitoring use Wire entities.

**Q: Why multiple snippets in one result?**  
A: Matching passages from the same document are grouped.

**Q: When are multiple queries worth it?**  
A: The server deduplicates across queries, so you are not charged for duplicate records.

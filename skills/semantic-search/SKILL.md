---
name: semantic-search
description: Retrieve quoteable document passages by meaning for financial research, evidence gathering, and thematic scans. Operations are semantic_search and semantic_search_multi.
---

# Semantic search

Semantic search retrieves indexed document passages ranked by embedding cosine similarity. Unlike Wire (headline + metadata), each result is one **document** with ordered **snippets** you can cite, plus title, scores, entity names, and source provenance.

| Operation | Role |
|-----------|------|
| `semantic_search` | Single query → documents packed under a `max_tokens` budget |
| `semantic_search_multi` | 2–8 queries → round-robin chunk consolidation, then the same document packing |

---

## When to use

**Use semantic search when you need to:**
- Pull evidence passages (numbers, quotes, guidance language)
- Research a theme hard to express as exact filters
- Cover a company or event from multiple angles (`semantic_search_multi`)
- Backtest what the corpus said in a time window
- Filter by publisher, snippet sentiment, utility, or coarse entity types

**Do not use semantic search when you need:**
- What just broke / novelty clusters → Wire
- Exact company-name monitoring → Wire entities or `retrieve_entity`
- Guaranteed entity match — semantic match is by meaning, not registry id

---

## Use cases

**Evidence for a specific claim** — Pull quoteable passages that support a narrow factual question (guidance, numbers, language).
```json
{
  "query": "STMicroelectronics AI data center revenue guidance 2026 2027",
  "max_tokens": 2000,
  "start_timestamp": "2026-07-01T00:00:00Z",
  "min_utility_score": 0.7
}
```

**Thematic macro / geopolitics** — Find passages about a theme even when tickers and headlines don’t say it cleanly.
```json
{
  "query": "Red Sea Houthi tanker disruption Brent crude shipping",
  "max_tokens": 3000,
  "start_timestamp": "2026-07-01T00:00:00Z",
  "min_cosine_similarity": 0.7
}
```

**Positive earnings reaction language** — Screen for upbeat beat-and-raise style coverage.
```json
{
  "query": "company beat earnings estimates raised guidance",
  "max_tokens": 5000,
  "start_timestamp": "2026-07-20T00:00:00Z",
  "min_sentiment_score": 0.4
}
```

**Negative tone screen** — Find passages stressing cost, disruption, or margin pressure.
```json
{
  "query": "airline fuel cost Middle East disruption margin pressure",
  "max_tokens": 1000,
  "start_timestamp": "2026-07-01T00:00:00Z",
  "max_sentiment_score": -0.3
}
```

**Publisher-constrained research** — Same research question, limited to publishers you trust.
```json
{
  "query": "Apple services revenue growth China",
  "max_tokens": 2000,
  "source_metadata_filters": [
    { "key": "publisher", "include_any": ["seekingalpha.com", "www.cnbc.com"] }
  ],
  "start_timestamp": "2026-01-01T00:00:00Z"
}
```

**Multi-angle company deep-dive** — Cover several facets of one company in one call instead of repeating single searches.
```json
{
  "queries": [
    "Tesla Q2 automotive margins",
    "Tesla Robotaxi Cybercab rollout",
    "Tesla Optimus humanoid robot guidance"
  ],
  "max_tokens": 5000,
  "start_timestamp": "2026-07-01T00:00:00Z",
  "min_utility_score": 0.5
}
```

**Multi-query banking theme** — Cover NIM, credit quality, and the Fed rate path in one call (always round-robin across queries).
```json
{
  "queries": [
    "bank net interest margin outlook",
    "regional bank credit quality deposits",
    "Federal Reserve rate path bank earnings"
  ],
  "max_tokens": 8000,
  "start_timestamp": "2026-07-01T00:00:00Z"
}
```

**Backtest window** — See what the corpus said during a fixed period, not the whole default history.
```json
{
  "query": "bank net interest margin outlook",
  "max_tokens": 3000,
  "start_timestamp": "2026-07-20T00:00:00Z",
  "end_timestamp": "2026-07-22T00:00:00Z"
}
```

---

## What you get back

Each hit is a **document**, not a raw chunk. Matching passages are in `snippets`.

| Field | Notes |
|-------|-------|
| `snippets` | Ordered passage texts from this document — primary value |
| `snippet_cosine_similarities` | Parallel to `snippets` (full detail only). Document rank = `max(...)` |
| `o200k_base_n_tokens` | Token count of returned snippets (full detail only) |
| `utility_score` | 0–1. Default filter floors at **0.4** |
| `sentiment_score` | −1..1 document-level tone (not entity-mention sentiment like Wire) |
| `canonical_entity_names` | Can be empty even when the text discusses a company |
| `document_id` | One result per document |
| `timestamp` | Used for time filters |
| `source` | `url`, `source_table` (`news`, `youtube`, `x_news`, …), `source_metadata` (`title`, `publisher`, …) |

Prefer specific queries + modest `max_tokens` (2000–5000 for single). Quote from `snippets`.

---

## Response detail

Both operations accept `detail`. **Default is `full`.** Reduced levels **drop keys** only — surviving fields keep the same names and paths as `full`. Pricing does not change with `detail`; this is about **response token cost** in the agent context.

| Level | What you get | Rough token savings vs `full` |
|-------|----------------|-------------------------------|
| `essential` | `title` + `snippets` + source timestamp | highest savings |
| `standard` | essential + citable source (`source_table`, `source_id`, `url`, and `source_metadata.title` when present) | medium |
| `full` | Every field (scores, entity names, document id, snippet similarities, token count, full source metadata, …) | — |

**Prefer `detail: "standard"`** for normal agent calls (enough to cite). Use `essential` when stuffing many hits into context; use `full` only when you need scores, entity names, or full source metadata.

```json
{
  "query": "STMicroelectronics AI data center revenue guidance 2026 2027",
  "max_tokens": 2000,
  "detail": "standard"
}
```

---

## Defaults that shape every call

| Param | Default | Implication |
|-------|---------|-------------|
| `detail` | `full` | Prefer `standard` (see **detail** above). |
| `start_timestamp` | `2026-01-01T00:00:00Z` | Not “all history.” Set explicitly for recent work or true backtests. |
| `min_utility_score` | `0.4` | Raise (e.g. 0.7) for denser results |
| `max_tokens` (single) | `2000`, ge 1000, le 20000 |
| `max_tokens` (multi) | `5000`, ge 1000, le 50000 | Require ≥500 tokens per query. |

Future `end_timestamp` is treated as realtime (no upper bound).

---

## How to write queries

1. **Specific topical phrases beat bare names.** `Tesla Q2 earnings margin guidance Robotaxi` → cosine ~0.77–0.80. Query `Tesla` alone → ~0.69 on generic pieces.
2. **Keyword-dense topic packs work well.** `Red Sea Houthi oil tanker Brent crude` slightly outperformed a long natural-language question on the same theme.
3. Questions are fine when they encode entities + mechanism; fluff words rarely help.
4. **Facet the problem for multi.** Split “margins / Robotaxi / Optimus” into separate queries.
5. Not an entity resolver. A query mentioning easyJet can still return Ryanair fuel passages — embedding neighborhood, not registry match.

---

## Filters

| Filter | Notes |
|--------|-------|
| Time | Inclusive on snippet `timestamp`. Default lower bound is 2026-01-01. |
| `min_cosine_similarity` | Post-filter. Aggressive floors (e.g. 0.75) easily return `[]` when top hits sit at ~0.72. Treat ~0.70 as a practical solid band for broad macro queries. |
| `min_utility_score` | Default 0.4. Raise for denser earnings/guidance language. Not Wire `importance_score`. |
| `min_sentiment_score` **or** `max_sentiment_score` | Mutually exclusive. Snippet/document-level tone. |
| `entity_types` | Coarse overlap (`COMPANY`, `COMMODITY`, …). Does not pin a named company. |
| `source_metadata_filters` | Publisher only. Closed **exact** domain strings (`seekingalpha.com`, `www.cnbc.com`). Not Wire’s free-form publisher names. `include_any` / `exclude_any`. |

---

## semantic_search_multi

Required: `queries` with min 2, max 8. One call, server-side dedupe, always **round-robin** across queries.

Returned documents are ordered by each document’s best snippet similarity. `max_tokens / len(queries)` must be at least 500.

---

## Workflow

| User intent | Approach |
|-------------|----------|
| “Find evidence that X said Y” | Specific query + raised `min_utility_score`; cite `snippets` + URL |
| “Research company across themes” | `semantic_search_multi` with facet queries |
| “What explained the oil move?” | Theme keywords; optional `COMMODITY` / `COUNTRY` entity types |
| “Only Seeking Alpha / CNBC” | `source_metadata_filters.include_any` with schema enum publishers |
| “What did the corpus say that week?” | Tight timestamp window — don’t rely on default start |
| “Realtime headline triage” | Wire first; escalate here for body evidence |

1. Decide Wire vs semantic search (headlines/novelty vs passage evidence).
2. Write a specific query (or 2–5 facet queries for multi). Set `start_timestamp` explicitly.
3. Tighten with `min_utility_score`, optional cosine floor ~0.70, publisher/entity_types only when needed.
4. Quote from `snippets` with URL from `source.url`.
5. For live story / who printed first, switch to Wire with canonical names from chunk entities.

---

## Limitations

- Passage retrieval, not novelty-aware news clustering.
- Not an exact entity filter; related peers/themes bleed in.
- `canonical_entity_names` can be incomplete vs the text.
- Publisher filter is a closed enum of domain-style strings — different from Wire publishers.
- Default `start_timestamp` and `min_utility_score` silently shape every naive call.
- Aggressive `min_cosine_similarity` easily returns `[]`.
- Corpus mixes news, YouTube transcripts, and social — transcripts are noisy but sometimes uniquely informative.

---

## FAQ

**Q: Why are Fed-rate results from months ago when I wanted “now”?**  
A: Default `start_timestamp` is 2026-01-01; ranking is pure cosine, not recency. Set a recent `start_timestamp`.

**Q: Why did `min_cosine_similarity: 0.75` return nothing?**  
A: Top hits may sit at ~0.72–0.73. Lower the floor or improve the query.

**Q: I searched easyJet and got Ryanair — bug?**  
A: Semantic neighborhood for airline + fuel. For name-locked monitoring use Wire entities.

**Q: Why multiple snippets in one result?**  
A: Matching passages from the same document are grouped. Join or cite individually as needed.

**Q: Wire `publisher: "bloomberg"` vs semantic `www.cnbc.com`?**  
A: Different vocabularies. Semantic publisher filter only accepts its schema enum.

**Q: When is multi worth it?**  
A: ≥2 distinct facets. Prefer multi over parallel single calls for dedupe.

**Q: Does round-robin scramble the final order?**  
A: Selection is diversified across queries; the response list is still sorted by best snippet cosine.

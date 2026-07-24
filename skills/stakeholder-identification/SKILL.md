---
name: stakeholder-identification
description: Map a market situation to ranked public-company winners and losers with verified tickers, signed impact scores, cited quotes, and a factor comparison matrix. Operation is identify_stakeholders.
---

# Stakeholder identification

`identify_stakeholders` answers “who matters for this situation, and why?” It retrieves finance-specific evidence, returns structured public-company impacts with cited quotes, verifies tickers, and ships a factor comparison matrix.

| Operation | Role |
|-----------|------|
| `identify_stakeholders` | Situation → ranked stakeholders + factor matrix |

**Latency:** commonly ~90 seconds; often 1–2+ minutes. Budget for that. Do not treat timeouts as “empty universe.” Do not spam parallel calls.

---

## When to use

**Use when you need:**
- Ranked public-company winners/losers for a market situation, with direction + magnitude
- Why each name is exposed (rationale) plus auditable quotes
- A factor matrix to compare the returned names at a glance
- Backtest-safe identification as-of a historical `end_timestamp`

**Do not use when you need:**
- Fast headline triage → Wire
- Quoteable document chunks → `semantic_search`
- One company’s live dossier → `retrieve_entity`
- Already-stored development packages → `search_developments` (SI creates fresh assessments; developments store prior ones)
- Non-company stakeholders (countries, commodities, people, private cos) → out of scope (`PUBLIC_COMPANY` only)

Generic LLMs invent tickers; this endpoint retrieves evidence, cites exact quotes, verifies symbols, and returns a reusable factor table — with optional point-in-time mode.

---

## Use cases

**Realtime supply shock** — Map who wins and loses if a concrete market shock hits, with verified tickers and cited reasons.
```json
{
  "query": "Strait of Hormuz near-total closure and oil supply shock"
}
```

**Realtime with steering** — Same exposure map, but steer coverage toward second-order names and keep the list tight.
```json
{
  "query": "Tesla Q2 2026 earnings miss and margin compression",
  "context": "EPS well below consensus; operating margin ~1.4%; negative FCF; energy storage strong.",
  "instructions": "Prioritize clear second-order exposures (EV peers, suppliers, AI compute). Prefer fewer high-confidence names."
}
```

**Historical / backtest** — Ask who mattered as of a past date, without live-web leakage after that cutoff.
```json
{
  "query": "Apple CEO succession Tim Cook to John Ternus",
  "end_timestamp": "2026-04-25T00:00:00Z"
}
```

**Constrained universe** — Force a short, sector-balanced list instead of a long undifferentiated dump.
```json
{
  "query": "who benefits from rising oil prices",
  "instructions": "Return at most 6 public companies spanning upstream, midstream, and airlines as losers."
}
```

**Regulation / tariffs** — Identify listed companies with clear regulatory exposure for a policy action.
```json
{
  "query": "EU Digital Markets Act fines on Big Tech self-preferencing",
  "instructions": "Focus on listed platforms and ad-tech peers with clear regulatory exposure."
}
```

---

## Params

| Param | Role |
|-------|------|
| `query` (required) | The situation. Non-empty after strip; blank → 422. |
| `context` | Extra facts about the situation (what happened, numbers, constraints) |
| `instructions` | How to shape the answer (coverage, count, sectors to prefer/avoid) — not treated as the search subject |
| `end_timestamp` | Omit/null = **realtime**. Past = **backfill**. Future → realtime. No `start_timestamp`. |

Prefer a concrete situation over a vague theme. Put hard facts in `context`; put process prefs in `instructions` (`"Prefer fewer high-confidence names"`, `"At most 6 names"`). Instructions work — they change count and composition.

Only verified public tickers are returned as stakeholders.

---

## Realtime vs backfill

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Realtime** | `end_timestamp` null/future | Uses live web retrieval plus historical structured SI context |
| **Backfill** | past `end_timestamp` | No live web. Evidence from indexed corpus and historical SI context clamped to `end_timestamp` — backtest-safe |

Use backfill for “what would an analyst have concluded knowing only info ≤ T?” Use realtime for “given the live tape, who is in the blast radius?”

Returned tickers are verified; unverified names are dropped.

---

## What you get back

### `stakeholder_impacts[]`
- `stakeholder`: `{ name, stakeholder_type: "PUBLIC_COMPANY", symbol }`
- `impact_score` ∈ [-1, 1]
- `confidence_score` ∈ [0, 1]
- `rationale`
- `sources_with_quotes[]` — exact quotes with source metadata

Scan strongest positives and strongest negatives first.

### `factor_matrix[]`
Rows = situation-specific factors. Each has `name` (snake_case), `description`, `factor_scale`, `rubric`, optional `allowed_values`, and `stakeholder_assessments` keyed by **symbol**.

Scales: `BINARY` | `CATEGORICAL` | `CONTINUOUS` [0,1] | `SIGNED_CONTINUOUS` [-1,1].

Use the matrix for “who is more exposed on dimension X?” Factor names are meant to be reusable across similar situations. A factor row may occasionally omit a symbol — treat missing cells as unknown, not zero.

---

## Workflow

| User intent | Approach |
|-------------|----------|
| “Who wins/loses if X?” | Realtime SI on a precise query; brief from impact extremes + factor matrix |
| “As of date D, who mattered?” | Same query + past `end_timestamp` |
| “Keep it short / sector-focused” | Put constraints in `instructions` |
| “Drill one ticker after SI” | `retrieve_entity`, Wire, or developments with `stakeholder_symbol` |
| “Just give me headlines” | Wire — do not pay SI latency for firehose |

1. Confirm the user wants situation → public-company exposure.
2. Craft a specific `query`; add `context` / `instructions` only when they sharpen the job.
3. Choose realtime vs backfill. Warn about ~1–2 min wait.
4. Call once; wait. On timeout, retry once — do not fan out parallel SI calls.
5. Brief: top ± impacts → factor matrix highlights → cite 1–2 quotes per key name.
6. Offer follow-ups via Wire / developments / `retrieve_entity` / semantic search using returned symbols.

---

## Limitations

- Slow — plan for ~90s; avoid speculative retries and fan-out.
- Public companies only; listing suffixes vary (`SMSN.IL`, `BYDDF`, `AMKBY`).
- Verification can drop proposed names — final list ⊆ verified survivors.
- Not a price target or trading signal — impact scores are reasoned exposures with citations.
- Backfill quality depends on indexed corpus ≤ T.
- Factor matrix is built per call — read names/rubrics; don’t assume identical every time.
- Does not return countries/commodities as stakeholders even when they dominate the narrative.

---

## FAQ

**Q: Why is this better than asking a normal agent?**  
A: Retrieval + quote binding + ticker verification + factor matrix + backtest mode.

**Q: Future `end_timestamp` — historical or live?**  
A: Treated as realtime.

**Q: Can I get private cos / OPEC / Brent as stakeholders?**  
A: Not as structured stakeholders. They may appear in rationales/sources only.

**Q: How do I continue after SI?**  
A: Take symbols → Wire filters, `search_developments(stakeholder_symbol=…)`, `retrieve_entity`, or semantic search for passages.

**Q: Empty / whitespace query?**  
A: 422 `query must be a non-empty string.`

---
name: signals
description: Daily market signal bars for a sector, country, or asset class. Operation is market_signal.
---

# Signals

`market_signal` aggregates Wire activity into contiguous daily or multi-day bars. Each bar carries wire-item attention and the net sentiment of entity mentions for one market scope.

| Operation | Role |
|-----------|------|
| `market_signal` | Attention and net sentiment bars for one sector, country, or asset class |

---

## When to use

**Use `market_signal` when you need to:**
- Track how much Wire coverage a market scope receives over time
- Measure the aggregate direction and magnitude of entity-mention sentiment
- Backtest attention and sentiment over a historical window
- Roll daily activity into coarser multi-day bars

**Do not use when you need:**
- The headlines behind a move → `list_wire`
- PIT or utility-filtered corpus passages → `semantic_search`
- A company dossier or entity-specific sentiment → `retrieve_entity`
- Ticker-level impact narratives → developments and events

---

## Use cases

**Sector activity** — Read the latest 14 bars of information-technology attention and net sentiment.
```json
{
  "value": "information_technology"
}
```

**Original energy coverage** — Exclude syndication duplicates when measuring energy activity.
```json
{
  "value": "energy",
  "novelty": ["new", "update", "correction"]
}
```

**Historical country signal** — Backtest US coverage over an explicit system-time window.
```json
{
  "value": "us",
  "start_timestamp": "2026-07-15T00:00:00Z",
  "end_timestamp": "2026-08-01T00:00:00Z"
}
```

**Weekly asset-class bars** — Roll crypto activity into stable 7-day buckets.
```json
{
  "value": "crypto",
  "resolution": "7d",
  "start_timestamp": "2026-07-01T00:00:00Z"
}
```

---

## What you get back

| Field | Notes |
|-------|-------|
| `bars` | Oldest → newest; empty buckets are present, never omitted |
| `bar` | Exclusive UTC bucket end |
| `count` | Distinct Wire items about the scope |
| `sentiment` | Net sum of every entity-mention sentiment on those items; `0` when empty or when no mentions exist |
| `sentiment_std` | Population standard deviation across those mention sentiments; `null` when there are no mentions |
| `partial` | `true` when the latest bucket is still in progress |
| `resolution` | Bucket length used |
| `methodology_version` | Signal definition version |

Floats are rounded to 2 decimal places. History is capped at 60 days; omitting `start_timestamp` returns the last 14 bars.

---

## Scope values

`value` is a single string; the dimension is inferred:

| Kind | Examples |
|------|----------|
| Sector | `information_technology`, `energy`, `financials`, `health_care` |
| Asset class | `equities`, `fixed_income`, `currencies`, `commodities`, `crypto`, `real_estate` |
| Country | Lowercase ISO alpha-2, such as `us`, `cn`, or `gb` |

`novelty` defaults to all four classes (`new`, `update`, `duplicate`, `correction`). Narrow it when you want originals/updates only.

---

## Workflow

| User intent | Approach |
|-------------|----------|
| “Is tech coverage increasing?” | Call with `value: information_technology`; read `count` over time |
| “What is the direction of energy sentiment?” | Call with `value: energy`; read the sign and magnitude of `sentiment` |
| “How has US attention changed?” | Call with `value: us` and an explicit window |
| “Show weekly crypto activity” | Call with `value: crypto`, `resolution: 7d` |

1. Pick one scope `value` (sector, country, or asset class).
2. Omit `start_timestamp` for the latest 14 bars, or set an explicit historical window.
3. Read `count` as coverage and `sentiment` as unnormalized net mention tone; use `sentiment_std` to assess disagreement.
4. For narrative detail, follow up with Wire using the same scope filters.

---

## Limitations

- One scope value per call — no multi-value baskets in a single request.
- Daily (or multi-day) bars only — no intraday resolution yet.
- Sentiment is an unnormalized net sum across entity mentions; magnitude rises with mention volume and should be compared primarily within the same scope over time.
- `count` is Wire items, not entity mentions, so it cannot be used to derive average mention sentiment.
- Country codes are free-form two-letter strings; invalid codes may return empty bars rather than a vocabulary error.
- Max lookback 60 days.

---

## FAQ

**Is `sentiment` an average?** No. It is the net sum of entity-mention sentiments. Positive and negative mentions offset each other; larger absolute values reflect direction plus mention volume.

**Why is `sentiment` 0 on a quiet day?** Empty or mention-less bars use `0` for this sum-type statistic. Check `count` and `sentiment_std` for context.

**What does `bar` mean?** Exclusive bucket end in UTC. A `1d` bar ending `2026-08-11T00:00:00Z` covers `[2026-08-10, 2026-08-11)`.

**Can I pass a company name?** No — use sectors, asset classes, or country codes. For a company, use Wire or `retrieve_entity`.

**How do I backtest?** Set `start_timestamp` / `end_timestamp` (system timestamps). `end_timestamp: null` means realtime.

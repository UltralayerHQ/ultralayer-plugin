---
name: alerts
description: Turn any alertable Ultralayer endpoint into a realtime monitor that emails, telegrams, or webhooks new results. Operations are create_alert, get_alerts, execute_alert, activate_alert, deactivate_alert, and delete_alert.
---

# Alerts

Alerts wrap an existing Ultralayer endpoint, poll it on a schedule, skip what was already delivered, and push new matches to email, Telegram, and/or a webhook. Same filters and queries as the underlying endpoint — plus schedule and delivery. Alerts are billed as the underlying endpoint calls — there is no alert surcharge.

| Operation | Role |
|-----------|------|
| `create_alert` | Validate endpoint + receivers (delivers a sample), then persist |
| `get_alerts` | List alerts with recent attempts and deliveries |
| `activate_alert` / `deactivate_alert` | `active` ↔ `paused` |
| `execute_alert` | Run now (same path as scheduled execution) |
| `delete_alert` | Permanent delete |

---

## When to use

**Use alerts when you need:**
- Continuous monitoring of Wire, developments, filings, events, or any other alertable path
- Delivery to inbox, Telegram, or a webhook without running your own cron
- The same conditions you already validated interactively, left running

**Do not use alerts when you need:**
- A one-shot answer → call the underlying endpoint directly
- To alert on endpoints that are not in the alertable set below

---

## Use cases

**Company wire monitor** — Get emailed when a company prints material news with strong positive or negative tone, skipping copycat syndication.
```json
{
  "name": "Apple important news",
  "path": "/v0/search/list_wire",
  "arguments": {
    "entities": ["Apple Inc."],
    "novelty": ["new", "update", "correction"],
    "entity_relevance": ["primary", "significant"],
    "min_entity_sentiment": 0.3,
    "max_entity_sentiment": -0.3,
    "detail": "full"
  },
  "interval_minutes": 15,
  "threshold": 1,
  "receivers": [{ "type": "email" }]
}
```
Sentiment min/max together = strong positive **or** strong negative. Novelty drops syndication duplicates.

**Semantic Monitoring** — Watch a theme by meaning (not just a ticker) and push new matching passages to email and a webhook.
```json
{
  "name": "Hormuz Oil Passages",
  "path": "/v0/search/semantic_search",
  "arguments": {
    "queries": ["Strait of Hormuz tanker disruption Brent crude"],
    "max_tokens": 1000,
    "min_utility_score": 0.5,
    "min_cosine_similarity": 0.75,
    "detail": "standard"
  },
  "interval_minutes": 30,
  "receivers": [
    { "type": "email" },
    { "type": "webhook", "url": "https://hooks.example.com/ultralayer" }
  ]
}
```

**Ticker development feed** — Get pinged when important new developments land for a ticker, with who is affected already scored.
```json
{
  "name": "Tesla Developments",
  "path": "/v0/search/search_developments",
  "arguments": {
    "stakeholder_symbol": "TSLA",
    "min_importance_score": 0.7,
    "limit": 20,
    "detail": "full"
  },
  "interval_minutes": 30,
  "receivers": [{ "type": "telegram", "chat_id": "123456789" }]
}
```

**Tracked event lifecycle** — Follow one known story (earnings cycle, M&A, etc.) and get notified as new developments attach to it.
```json
{
  "name": "Tesla 2026 Financial Event",
  "path": "/v0/search/retrieve_event_developments",
  "arguments": {
    "event_id": 125,
    "limit": 20,
    "include_future_developments": true,
    "detail": "standard"
  },
  "interval_minutes": 60,
  "lookback_minutes": 180,
  "receivers": [{ "type": "email" }]
}
```
Discover `event_id` via `search_events` or a development hit first.

**Events scan (M&A)** — Stay on top of new deal activity without polling for mergers yourself.
```json
{
  "name": "M&A Events",
  "path": "/v0/search/search_events",
  "arguments": {
    "event_type": "merger_acquisition",
    "developments_per_event": 1,
    "limit": 10,
    "detail": "essential"
  },
  "interval_minutes": 60,
  "receivers": [{ "type": "email" }]
}
```

**Guidance raised or cut** — Across the market, get emailed when a company revises an official outlook up or down.
```json
{
  "name": "Guidance raised or cut",
  "path": "/v0/filings/list_guidance",
  "arguments": {
    "revision_direction": ["raised", "lowered"],
    "detail": "standard",
    "limit": 10
  },
  "interval_minutes": 60,
  "receivers": [{ "type": "email" }]
}
```

**KPI definition and disclosure** — Across the market, get pinged when a release changes how a metric is calculated, or starts or stops highlighting a named KPI.
```json
{
  "name": "KPI definition and disclosure",
  "path": "/v0/filings/list_changes",
  "arguments": {
    "change_categories": ["metric_definition", "disclosed_metric"],
    "detail": "standard",
    "limit": 10
  },
  "interval_minutes": 60,
  "receivers": [{ "type": "email" }]
}
```

**Guide versus actual** — Get emailed when a prior guide significantly exceeds or significantly misses the reported result.
```json
{
  "name": "Guidance versus actual",
  "path": "/v0/filings/list_guidance_outcomes",
  "arguments": {
    "outcomes": ["significantly_exceeded", "significantly_missed"],
    "detail": "standard",
    "limit": 10
  },
  "interval_minutes": 60,
  "receivers": [{ "type": "email" }]
}
```

---

## Alertable endpoints

| `path` | What you’re monitoring |
|--------|------------------------|
| `/v0/search/list_wire` | Filtered headlines |
| `/v0/search/semantic_search` | PIT / utility-filtered corpus passages |
| `/v0/search/search_developments` | Structured developments |
| `/v0/search/search_events` | Events with recent activity |
| `/v0/search/retrieve_event_developments` | Developments on a known `event_id` |
| `/v0/filings/list_guidance` | Official outlook series that moved in the window |
| `/v0/filings/list_changes` | Wording changes in the window |
| `/v0/filings/list_guidance_outcomes` | Company guidance versus the reported actual |

Wire for headlines. Developments/events for structured lifecycle. Thematic / corpus passage monitors use path `/v0/search/semantic_search`. `list_guidance` for the official outlook and revision. `list_changes` for wording changes between releases. `list_guidance_outcomes` for guide versus actual.

---

## Create params

| Field | Notes |
|-------|-------|
| `name` | Optional; unique per user. Duplicate → 400. |
| `path` | One of the alertable paths above |
| `arguments` | Upstream body **minus timestamps**. Never include `start_timestamp` / `end_timestamp` (422). |
| `interval_minutes` | ≥ 5 and a multiple of 5 |
| `lookback_minutes` | Rolling window; defaults to `interval_minutes` |
| `threshold` | Min matching items before delivery (default 1) |
| `receivers` | 1–2 targets; duplicate targets rejected |

### Receivers

| Type | Config |
|------|--------|
| `email` | `{ "type": "email" }` — delivers to the verified account email |
| `telegram` | `{ "type": "telegram", "chat_id": "…" }` — start the Ultralayer alerts bot first |
| `webhook` | `{ "type": "webhook", "url": "…", "headers"?, "body_wrap_key"?, "body_wrap_as_string"?, "body_extra"? }` — public http(s) only |

Canonical envelope: `{ "name": "<alert name>", "response": <exact upstream JSON> }`. Email/telegram always get that envelope. Subject: `ULTRALAYER ALERT: {name}`.

`create_alert` immediately calls the endpoint and delivers a **sample** to every receiver before persisting.

#### Webhook body shaping

Use these when the receiving API rejects an arbitrary JSON body and requires a fixed envelope. Structured merge only — no templating.

| Field | Role |
|-------|------|
| `body_wrap_key` | Place the envelope under this key. One level of dot-separated nesting (`variables.PAYLOAD` → `{"variables": {"PAYLOAD": …}}`). |
| `body_wrap_as_string` | JSON-encode the envelope to a string before placing it. Requires `body_wrap_key`. |
| `body_extra` | Static keys merged into the top level of the body. Values must be string / number / boolean / null. |

Examples:

```json
{
  "type": "webhook",
  "url": "https://api.example.com/hooks/ingest",
  "headers": { "Authorization": "Bearer …" },
  "body_wrap_key": "client_payload",
  "body_extra": { "event_type": "ultralayer_alert" }
}
```
→ `{ "event_type": "ultralayer_alert", "client_payload": { "name": "…", "response": … } }`

```json
{
  "type": "webhook",
  "url": "https://ci.example.com/api/v4/projects/1/trigger/pipeline",
  "body_wrap_key": "variables.PAYLOAD",
  "body_wrap_as_string": true,
  "body_extra": { "token": "…", "ref": "main" }
}
```
→ `{ "token": "…", "ref": "main", "variables": { "PAYLOAD": "{\"name\":…}" } }`

Arbitrary-JSON receivers (Zapier, Make, n8n, custom endpoints) need none of these fields.

---

## Delivery behavior

Each run checks a rolling lookback window and never re-sends data already delivered. The alert fires only when at least `threshold` new items match. Attempt outcomes (`delivered`, `no_data`, `below_threshold`, `endpoint_error`, `delivery_partial`, `delivery_failed`) and recent deliveries are visible via `get_alerts`. If all receivers fail, the same data is retried next run.

There is no update-alert endpoint. Changing filters = delete + recreate.

---

## Workflow

1. Prototype filters with the underlying endpoint until results look right.
2. `create_alert` with those arguments (no timestamps) + receivers + interval. Expect a sample delivery.
3. `get_alerts` to confirm status and early attempts.
4. `execute_alert` to force a test run. `deactivate_alert` / `activate_alert` to pause/resume.
5. `delete_alert` when done. Prefer pause over delete if the config may be reused.

---

## Limitations

- Poll-based (minimum 5-minute cadence), not push-on-insert.
- Only the alertable paths listed above.
- Max 2 receivers; email is account-bound.
- Webhooks must be public.
- Webhook `headers` are count- and size-capped, and framing, routing and provenance headers (`Host`, `Content-Type`, `Content-Length`, `User-Agent`, `X-Forwarded-*`, …) are rejected. `Authorization`, `Cookie` and custom `X-` headers are fine.
- Webhook `body_extra` and `body_wrap_key` are count-, size- and character-capped — see the OpenAPI schema for exact limits. Values are scalars only, and a field name containing a literal `.` cannot be targeted by `body_wrap_key`.
- Create always sends a sample — warn before creating noisy webhooks.
- No patch API for existing alerts.

---

## FAQ

**Q: Is this more expensive than calling the API myself on a cron?**  
A: Same underlying pricing; you gain dedupe, thresholding, multi-channel delivery, history, and managed scheduling.

**Q: Why can’t I set `start_timestamp` in arguments?**  
A: The alert owns the rolling window. Including timestamps → 422.

**Q: Why did create email me immediately?**  
A: Receiver verification delivers a real sample by design.

**Q: `no_data` every 15 minutes — broken?**  
A: Healthy. The window had zero matches. Check `deliveries` for real fires.

**Q: How do I change an alert’s filters?**  
A: There is no patch. Pause or delete + recreate.

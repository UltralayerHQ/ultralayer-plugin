---
name: alerts
description: Realtime alerts. Say once what you want to watch (a company you hold, a commodity, a country, an outlook change) and every new match is sent to your email, Telegram, or a webhook as soon as it lands. Operations are create_alert, get_alerts, activate_alert, deactivate_alert, and delete_alert.
---

# Alerts

Alerts watch the market for you. Describe once what you care about, such as a company you hold, a commodity, a country, or a kind of event, and Ultralayer sends you each new item that matches the moment it arrives. There is nothing to refresh and nothing to run on your own computer.

Alerts arrive by email, on Telegram, or at a webhook, which connects them to your own tools such as Zapier, a spreadsheet, or a trading system. Each one carries the item itself with its source, so you can see what happened and where it came from.

### What you can watch

| Watch | What you get | Example |
|-------|--------------|---------|
| News | New headlines about the companies, commodities, countries, or sectors you pick | "Tell me when there is important news about Apple" |
| Developments | Significant business events, with the companies they affect and how strongly | "Ping me when something material happens that hits Tesla hard" |
| Company outlooks | A company raises, cuts, or issues its official forecast | "Email me when a company in my portfolio raises or cuts its revenue outlook" |
| Results versus outlook | A company's results beat or miss what it told investors to expect | "Tell me when a company badly misses its own forecast" |
| Filing changes | A company changes what its earnings release says: a new risk, a dropped metric, a redefined KPI | "Tell me when Nvidia adds or drops a risk in its earnings release" |

### What it costs

An active alert costs $0.007 an hour, about $5 a month, plus a small charge for what it delivers: $0.001 per 10 news items, outlooks, results, or filing changes, and $0.01 per development. A paused alert costs nothing. If a delivery reaches none of your destinations, its charge is refunded.

### How to start

Ask your AI something like "Set up an alert that emails me when Apple has important news with a strongly positive or negative tone." It turns that into the alert's settings, sends a test message to confirm the delivery works, and starts watching. You can pause, resume, or delete an alert at any time.

---

## How alerts work (reference for agents)

An alert is a name, a `conditions` object, a `detail`, and one or two receivers. Ultralayer matches new rows as they land and delivers them. There is no poll interval, lookback, or threshold.

`conditions.source` is one of `wire`, `developments`, `guidance`, `guidance_outcomes`, `changes`. Within a list the values are OR. Across fields they are AND. Both a min and a max on the same score means the disjoint tails: `x >= min` OR `x <= max`.

Events, stock screening, semantic search, and `guidance_bias` are not alertable.

While an alert is active it costs 0.007 credits per hour (about $5 a month). Each delivery also costs the data: `ceil(n / 10) * 0.001` for wire, guidance, guidance outcomes, and changes, and `0.01 * n` for developments.

| Operation | Role |
|-----------|------|
| `create_alert` | Validate, charge the current hour, persist, return the alert |
| `get_alerts` | List alerts with recent attempts and deliveries |
| `activate_alert` / `deactivate_alert` | `paused` ↔ `active`. Activate resets the match clock and charges the hour if it is unpaid |
| `delete_alert` | Permanent delete |

---

## When to use

**Use alerts when you need:**
- Continuous monitoring of wire, developments, guidance, guidance outcomes, or disclosure changes
- Delivery to inbox, Telegram, or a webhook without running your own process

**Do not use alerts when you need:**
- A one-shot answer → call the underlying endpoint
- Events, screens, semantic search, or guidance bias

---

## Use cases

**Company wire monitor** — Email when a company prints material news with strong positive or negative tone. Duplicates are excluded by the default novelty list.
```json
{
  "name": "Apple important news",
  "conditions": {
    "source": "wire",
    "entities": ["Apple Inc."],
    "entity_relevance": ["primary", "significant"],
    "min_entity_sentiment": 0.3,
    "max_entity_sentiment": -0.3,
    "min_importance_score": 0.6
  },
  "detail": "full",
  "receivers": [{ "type": "email" }]
}
```

**Ticker developments** — Ping when a new development names the ticker with a strong impact. Symbols and impact are on the same stakeholder.
```json
{
  "name": "Tesla developments",
  "conditions": {
    "source": "developments",
    "symbols": ["TSLA"],
    "min_impact_score": 0.5,
    "min_importance_score": 0.7
  },
  "detail": "full",
  "receivers": [{ "type": "telegram", "chat_id": "123456789" }]
}
```

**Guidance raised or cut**
```json
{
  "name": "Guidance raised or cut",
  "conditions": {
    "source": "guidance",
    "revision_direction": ["raised", "lowered"]
  },
  "detail": "standard",
  "receivers": [{ "type": "email" }]
}
```

**KPI definition and disclosure**
```json
{
  "name": "KPI definition and disclosure",
  "conditions": {
    "source": "changes",
    "change_categories": ["metric_definition", "disclosed_metric"]
  },
  "detail": "standard",
  "receivers": [{ "type": "email" }]
}
```

**Guide versus actual**
```json
{
  "name": "Guidance versus actual",
  "conditions": {
    "source": "guidance_outcomes",
    "outcomes": ["significantly_exceeded", "significantly_missed"]
  },
  "detail": "standard",
  "receivers": [{ "type": "email" }]
}
```

---

## Create params

| Field | Notes |
|-------|-------|
| `name` | Optional; unique per user. Duplicate → 400. Default is the source name. |
| `conditions` | Discriminated on `source`. See the examples. Up to 50 symbols or entities. |
| `detail` | `full` (default), `standard`, or `essential`. Same projection as the read endpoint. |
| `receivers` | 1–2 targets; duplicate targets rejected. |

Two defaults apply when a field is left out: wire `novelty` is `["new", "update", "correction"]`, and changes `min_materiality_score` is 0.4. `get_alerts` shows the stored conditions with every field.

Wire `entities` are names or tickers, resolved once at create. Developments `symbols` are exact tickers. Guidance, outcomes, and changes symbols are uppercased.

Guidance `metric_keys` are a case-insensitive substring unless `metric_key_exact_match` is true. In `segments`, `"null"` means the consolidated (null) segment.

### Receivers

| Type | Config |
|------|--------|
| `email` | `{ "type": "email" }` — delivers to the verified account email |
| `telegram` | `{ "type": "telegram", "chat_id": "…" }` — start the Ultralayer alerts bot first |
| `webhook` | `{ "type": "webhook", "url": "…", "headers"?, "body_wrap_key"?, "body_wrap_as_string"?, "body_extra"? }` — public http(s) only |

Canonical envelope: `{ "name": "<alert name>", "response": [items] }`. Each item is what the matching read endpoint returns at `detail`. Email and telegram always get that envelope. Subject: `ULTRALAYER ALERT: {name}`.

`create_alert` delivers a sample with `"response": []` to every receiver before persisting, and charges the current hour. A 402 rejects the create.

#### Webhook body shaping

Use these when the receiving API rejects an arbitrary JSON body and requires a fixed envelope. Structured merge only — no templating.

| Field | Role |
|-------|------|
| `body_wrap_key` | Place the envelope under this key. One level of dot-separated nesting (`variables.PAYLOAD` → `{"variables": {"PAYLOAD": …}}`). |
| `body_wrap_as_string` | JSON-encode the envelope to a string before placing it. Requires `body_wrap_key`. |
| `body_extra` | Static keys merged into the top level of the body. Values must be string / number / boolean / null. |

Arbitrary-JSON receivers (Zapier, Make, n8n, custom endpoints) need none of these fields.

---

## Delivery behavior

A row matches only when it is new since the alert became active. Attempt outcomes are `delivered`, `delivery_partial`, `delivery_failed`, `no_data`, and `insufficient_credit`. They are visible via `get_alerts`. If every receiver fails, the data cost is refunded. Insufficient credit pauses the alert.

There is no update-alert endpoint. Changing conditions means delete and recreate.

---

## Workflow

1. Prototype the filters on the underlying read endpoint until the results look right.
2. `create_alert` with those conditions, a detail, and receivers. Expect a sample delivery and the hourly charge.
3. `get_alerts` to confirm status and later attempts.
4. `deactivate_alert` / `activate_alert` to pause and resume. Activate does not replay rows from before that moment.
5. `delete_alert` when done. Prefer pause over delete if the config may be reused.

---

## Limitations

- Sources: wire, developments, guidance, guidance outcomes, changes. Not events, screens, semantic search, or guidance bias.
- Max 2 receivers; email is account-bound.
- Webhooks must be public.
- Webhook `headers` are count- and size-capped, and framing, routing and provenance headers (`Host`, `Content-Type`, `Content-Length`, `User-Agent`, `X-Forwarded-*`, …) are rejected. `Authorization`, `Cookie` and custom `X-` headers are fine.
- Webhook `body_extra` and `body_wrap_key` are count-, size- and character-capped. Values are scalars only, and a field name containing a literal `.` cannot be targeted by `body_wrap_key`.
- Create always sends a sample — warn before creating noisy webhooks.
- No patch API for existing alerts.

---

## FAQ

**Q: Is this more expensive than calling the API myself on a schedule?**  
A: An active alert costs $0.007 an hour plus the data it delivers. In return you get matching within seconds of a row landing (not at your poll interval), no repeats, delivery to email, Telegram, or a webhook, a history of attempts and deliveries, and nothing to run yourself.

**Q: Why can’t I set `start_timestamp` or `end_timestamp`?**  
A: An alert only looks forward: it matches rows that arrive after it became active. Timestamps are not condition fields, so sending one is a 422.

**Q: Will I get older items when I create or resume an alert?**  
A: No. Create and activate both start the match clock at that moment. Nothing from before is replayed.

**Q: Why did create email me immediately?**  
A: Receiver verification delivers a sample with `"response": []` by design, so a bad receiver fails before you are charged.

**Q: No deliveries for hours — is it broken?**  
A: Usually not: an alert sends only when a new row matches, and narrow conditions can stay quiet for a long time. Check `get_alerts`. If the status is `paused` with an `insufficient_credit` attempt, top up and activate it again.

**Q: How do I change an alert’s filters?**  
A: There is no patch. Delete and recreate. Pause instead of delete if you may want the old one back.

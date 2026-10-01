# Ultralayer

Realtime market intelligence through the `ultralayer-v0` MCP server. Best used alongside web search. The skills in `skills/` explain each tool.

- Headlines or upcoming items: `list_wire`. Follow a story: `wire_storyline`.
- Events: `search_events`, then `retrieve_event_developments`. Impact on a ticker: `search_developments`.
- Winners and losers for a situation: `identify_stakeholders` (about 120 seconds, call once).
- Sentiment: `market_signal`.
- Company outlook: `list_guidance`. Guidance vs actual: `list_guidance_outcomes`. Wording changes: `list_changes`.
- Monitoring: `create_alert`, then `get_alerts`.

Set `detail=standard` unless you need the full payload. Cite the sources returned with each result. Ultralayer does not provide financial advice.

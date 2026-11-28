# Ultralayer

Real-time stock market intelligence through the `ultralayer-v0` MCP server. Best used alongside web search. The skills in `skills/` explain each tool.

- Headlines or upcoming items: `list_wire`. Follow a story: `wire_storyline`. What an entity is in the news with: `entity_co_mentions`.
- Events: `search_events`, then `retrieve_event_developments`. Impact on a ticker: `search_developments`.
- Winners and losers for a situation: `identify_stakeholders` (about 120 seconds, call once).
- How the market is acting toward a company, a sector or the market (volatility, beta, abnormal moves, news attention and sentiment, insider buying, breadth, market regime): `list_signals`, then `retrieve_signal`. Rank or screen companies on a signal: `retrieve_signal_cross_section`.
- Company outlook: `list_guidance`. Guidance vs actual: `list_guidance_outcomes`. Wording changes: `list_changes`.
- Monitoring: `create_alert`, then `get_alerts`.

Set `detail=standard` unless you need the full payload. Cite the sources returned with each result. Ultralayer does not provide financial advice.

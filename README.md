# Ultralayer

[![Glama](https://img.shields.io/badge/Glama-listed-blue)](https://glama.ai/mcp/connectors/io.github.UltralayerHQ/ultralayer)
[![MCP Registry](https://img.shields.io/badge/MCP%20Registry-listed-blue)](https://registry.modelcontextprotocol.io/?q=io.github.UltralayerHQ%2Fultralayer)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![smithery badge](https://smithery.ai/badge/ultralayer/ultralayer-v0)](https://smithery.ai/servers/ultralayer/ultralayer-v0)

Realtime financial context for AI agents: what changed, who is affected, and what to watch next. One suite covering news, events, guidance, filing changes, sentiment, stakeholders, and alerts. Information-efficient responses with evidence for every result. First-class point-in-time safety for backtests. Pairs well with web search and a market-data API. All data is our own.

## Links

- [Website](https://ultralayer.ai)
- [App](https://app.ultralayer.ai)
- [Docs](https://docs.ultralayer.ai)
- [Console](https://console.ultralayer.ai)

## What's included

- **MCP server** at `https://api.ultralayer.ai/v0/mcp` (OAuth/API key)
- **Agent skills** for using Ultralayer's market intelligence effectively
- **Highlighted capabilities include** market news that separates new information from repeats, developments with company impact scores, broader event timelines, company outlooks, disclosure changes, stakeholder analysis, and alerts

## Connect

Ultralayer supports both OAuth and API key. Choose how you connect.

**OAuth.** Sign in in the browser.

**API key.** Create a key at [console.ultralayer.ai](https://console.ultralayer.ai) and send it as a bearer token:

```json
{
  "mcpServers": {
    "ultralayer-v0": {
      "type": "http",
      "url": "https://api.ultralayer.ai/v0/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

## License

[MIT](LICENSE)

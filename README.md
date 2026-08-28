# Epsilon plugin

Trade and build on [Epsilon](https://www.epsilon.exchange) — the
limit-order DEX on Robinhood Chain — from your AI agent.

This plugin bundles:

- **MCP server** — 12 tools: quotes, token search, portfolio, and
  non-custodial limit / stop-loss / DCA order placement. Orders are signed
  locally; keys never leave your machine.
- **`epsilon-trading` skill** — teaches the agent the safe trading
  workflow: quote → approve → place → review, confirmation gates, spend
  caps, and scheduled position-check recipes.
- **`epsilon-api-dev` skill** — teaches the agent to integrate the Epsilon
  REST API and TypeScript/Python SDKs into bots and apps.
- **Robinhood Chain rules** — chain facts and conventions so code the agent
  writes for chain id 4663 lands on the right endpoints and signing scheme.

## One-command skill install (any skills-compatible agent)

```
npx skills add alienbase-xyz/epsilon-plugin
```

Installs the `epsilon-trading` and `epsilon-api-dev` skills into Cursor,
Claude Code, Codex, Cline, and 13+ other agent runtimes. Then tell your
agent to set up the MCP server (config below) for live tools.

## Claude Code marketplace

This repo is also a Claude Code plugin marketplace:

```
/plugin marketplace add alienbase-xyz/epsilon-plugin
/plugin install epsilon
```

## Setup

1. Get a free API key at
   [developers.epsilon.exchange/dashboard](https://developers.epsilon.exchange/dashboard)
   and set `EPSILON_API_KEY`.
2. (Trading only) Set `EPSILON_WALLET_KEY` to a **dedicated trading
   wallet's** private key — never your main wallet. Read-only tools work
   without it.
3. Safety rails are on by default in this plugin's MCP config:
   `EPSILON_MAX_ORDER_USD=250` (per-order cap) and
   `EPSILON_REQUIRE_CONFIRM=true` (the agent must show you a confirmation
   before any order). Adjust in your MCP settings.

## Without the plugin

- MCP directly: `npx -y @epsilon-exchange/mcp` · hosted read-only:
  `https://api.epsilon.exchange/mcp` (`X-API-Key` header)
- SDKs: [`@epsilon-exchange/sdk`](https://www.npmjs.com/package/@epsilon-exchange/sdk) (npm) ·
  `epsilon-exchange` (PyPI)
- Docs: [developers.epsilon.exchange](https://developers.epsilon.exchange) ·
  [API reference](https://developers.epsilon.exchange/api/) ·
  [llms.txt](https://developers.epsilon.exchange/llms.txt)

## Risk disclosure

Trading is at your own risk. Orders placed by an AI agent are your orders.
Use a dedicated wallet, keep the spend cap on, and review every
confirmation. Epsilon is non-custodial: funds stay in your wallet until a
keeper fills your signed order on-chain.

## License

MIT

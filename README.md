# Epsilon plugin

Trade on [Epsilon](https://www.epsilon.exchange) — the limit-order DEX on
Robinhood Chain — straight from your AI agent, in about 30 seconds:
install, paste a free API key, then say *"buy $20 of WETH if it dips 3%"*.
Non-custodial: orders are EIP-712 messages signed locally, your keys never
leave your machine, and funds stay in your wallet until fill.

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

In Cursor, the plugin prompts for these at install (Configure on the plugin
card); elsewhere set them as environment variables:

1. `EPSILON_API_KEY` (required) — free self-serve key from
   [developers.epsilon.exchange/dashboard](https://developers.epsilon.exchange/dashboard).
2. `EPSILON_WALLET_KEY` (trading only) — private key of a **dedicated
   trading wallet** with limited funds, never your main wallet. Leave empty
   for read-only mode (quotes, market data, portfolio).
3. `EPSILON_REFERRER` (optional) — a 0x address that earns the on-chain
   referral leg of fills you route (permissionless rev-share).

Safety rails are on by default in this plugin's MCP config:
`EPSILON_MAX_ORDER_USD=250` (per-order cap, fails closed) and
`EPSILON_REQUIRE_CONFIRM=true` (the agent must show you a preview and get
your confirmation before any order). Adjust in your MCP settings.

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

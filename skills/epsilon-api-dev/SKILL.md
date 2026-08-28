---
name: epsilon-api-dev
description: Integrate the Epsilon API/SDKs into an application — trading bots, DeFi apps, dashboards, or agents on Robinhood Chain. Use when writing code that quotes, places, or tracks Epsilon orders via the REST API, TypeScript SDK, or Python SDK (not when trading interactively via MCP tools).
---

# Building on the Epsilon API

Epsilon exposes a keyed REST API (`https://api.epsilon.exchange/v1`) plus
first-party SDKs that handle the hard part — local EIP-712 order signing —
so integrations never send private keys anywhere.

## Choose the layer

| Building | Use |
|---|---|
| Bot / backend in TS or JS | `@epsilon-exchange/sdk` (npm) |
| Bot / backend in Python | `epsilon-exchange` (PyPI) |
| Any other language | REST + OpenAPI (`https://developers.epsilon.exchange/openapi.json`) — but you must implement EIP-712 signing against the v7.1 struct; port from an SDK, don't improvise |
| AI agent runtime | MCP: `npx -y @epsilon-exchange/mcp` (see the epsilon-trading skill) |

## Setup facts

- Auth: `X-API-Key: eps_…` header. Free self-serve keys:
  https://developers.epsilon.exchange/dashboard (Privy login). Limits:
  free tier 120 quote / 600 read / 30 order per minute, `X-RateLimit-*`
  headers on every response.
- Chain: Robinhood Chain id **4663**, RPC
  `https://rpc.mainnet.chain.robinhood.com`, explorer
  `https://robinhoodchain.blockscout.com`.
- EIP-712 domain: `{ name: "EpsilonRouter", version: "7", chainId: 4663,
  verifyingContract: 0xdb41FA80016DC946cEB7B8512c3423463d3F260f }`. The
  Order struct includes `triggerAmountOut` (v7.1) — omitting it or
  hand-rolling the struct produces signatures the verifier rejects.
- Keep secrets in env vars (`EPSILON_API_KEY`, wallet key for signing).
  Never commit them; never log them.

## TypeScript happy path

The SDK is function-based: a shared `ApiConfig` plus `apiGet`/`apiPost`
helpers for endpoints, and dedicated functions for the signing-critical
paths.

```ts
import {
  apiGet, apiPost, fetchOrderRoute,
  buildAndSignOrder, calculateTriggerPrice, approveRouter,
  USDG_ADDRESS, DEFAULT_RPC_URL,
} from '@epsilon-exchange/sdk'
import { privateKeyToAccount } from 'viem/accounts'

const cfg = { baseUrl: 'https://api.epsilon.exchange', apiKey: process.env.EPSILON_API_KEY! }
const account = privateKeyToAccount(process.env.WALLET_KEY as `0x${string}`)

// 1. resolve the counter-token + fetch the ORDER route (feeLegs matter)
const { tokens } = await apiGet<{ tokens: { address: string; decimals: number }[] }>(
  cfg, '/v1/tokens?search=WETH')
const weth = tokens[0]
const amountIn = 100_000_000n // 100 USDG (6 decimals)
const route = await fetchOrderRoute(cfg, {
  tokenIn: USDG_ADDRESS, tokenOut: weth.address,
  amountIn: amountIn.toString(), slippagePpm: 10_000, wallet: account.address,
})

// 2. ensure router allowance once per token (skippable if already approved)
await approveRouter(DEFAULT_RPC_URL, account, USDG_ADDRESS, amountIn)

// 3. sign locally, submit the signature — the key never leaves the process
const signed = await buildAndSignOrder({
  account, tokenIn: USDG_ADDRESS, tokenOut: weth.address as `0x${string}`,
  amountIn, triggerPrice: calculateTriggerPrice('0.00025', weth.decimals),
  tokenInDecimals: 6, deadline: Math.floor(Date.now() / 1000) + 86_400,
  slippagePpm: 10_000, kind: 'limit', feeLegs: route.feeLegs,
})
const { orderHash } = await apiPost<{ orderHash: string }>(
  cfg, '/v1/orders', { orderType: 'limit', ...signed })
```

The Python SDK wraps the same surface in a client class:
`EpsilonClient(api_key=…)` with `search_tokens` / `quote` / `order_route` /
`submit_order` / `cancel_order` methods, plus the same free-standing
`build_and_sign_order` and `calculate_trigger_price`. Both SDKs are
golden-fixture-tested against the on-chain verifier — trust their output
over any manual re-derivation.

## Integration checklist

1. Quote before every placement; never reuse stale routes (`feeLegs` embed
   referral data and route freshness matters).
2. Check/raise the router allowance before the first order per token.
3. Persist `orderHash` from submission; poll `GET /v1/orders/{hash}` for
   state (`open → filled/cancelled/expired`), respecting rate limits —
   or use webhooks when available rather than tight polling.
4. Handle 401 (`KEY_MISSING`/`KEY_INVALID`), 429 (back off until
   `X-RateLimit-Reset`), and 4xx validation errors (the body's `code` +
   `error` are actionable — surface them).
5. For products routing OTHER people's flow: pass `referrer=0xYourAddress`
   on `GET /v1/route` (TS: `fetchOrderRoute({ …, referrer })`, Python:
   `order_route(…, referrer=…)`) — the referral leg of every fill you route
   is paid to that address on-chain, permissionlessly, from the first fill.
   The ppm is policy-set; you pick the destination only. Apply for the
   builder tier on the dashboard when you need 5× rate limits and the
   earnings view.

## Reference

- API reference: https://developers.epsilon.exchange/api/
- Machine-readable index: https://developers.epsilon.exchange/llms.txt
- OpenAPI: https://developers.epsilon.exchange/openapi.json
- Support: team@epsilon.exchange

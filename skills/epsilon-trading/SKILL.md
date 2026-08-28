---
name: epsilon-trading
description: Trade on Epsilon (Robinhood Chain DEX) through the Epsilon MCP tools — quotes, limit orders, stop-losses, DCA schedules, cancels, portfolio and order review. Use when the user wants to trade, check positions, or manage orders on Robinhood Chain, including scheduled/recurring position checks.
---

# Trading on Epsilon

Epsilon is a non-custodial limit-order DEX on Robinhood Chain. Orders are
signed locally with the configured wallet and executed on-chain by keepers —
funds never leave the wallet until fill. You interact through the Epsilon
MCP tools (`npx -y @epsilon-exchange/mcp`).

## Ground rules (read first)

1. **Money tools are gated.** `approve_token`, `place_limit_order`,
   `place_dca_order`, and `cancel_order` spend or commit funds. If
   `EPSILON_REQUIRE_CONFIRM` is on, they return a confirmation preview
   first — show it to the user verbatim and only re-call with
   `confirm: true` after explicit user approval. Never auto-confirm.
2. **Respect the spend cap.** If a placement fails the
   `EPSILON_MAX_ORDER_USD` check, tell the user the cap and stop. Do not
   split one intent into multiple orders to slip under a cap unless the
   user explicitly asks for that.
3. **Prices are token_out per 1 token_in.** Sanity-check direction with a
   quote before placing: for "sell WETH at 4000 USDG", `token_in` = WETH,
   `token_out` = USDG, `limit_price` = 4000.
4. **Quotes are the scarce resource** (tightest rate limit). Quote once per
   decision, not in a loop. Poll order state with `get_order_status` or
   `list_orders` instead.

## Workflow: place an order

1. `search_tokens` to resolve symbols → addresses (never guess addresses;
   ambiguous symbols exist).
2. `get_quote` (token_in, token_out, amount) to see the current market rate
   and route. Compare with the user's target price; if the order would fill
   immediately (limit already at/under market for a sell), say so.
3. `wallet_info` / `get_portfolio` to confirm balance covers the amount.
4. `approve_token` for token_in if allowance is short — the placement tool
   errors with the exact needed approval if you skip this.
5. `place_limit_order` — `stop_loss: false` for take-profit/limit
   semantics (fills at limit_price or better), `stop_loss: true` to
   trigger when price drops to limit_price. Set `expiry_hours`
   (default 168h; 0 = never expires — confirm the user really wants that).
6. Report the returned `orderHash`, expiry, and (for stop-losses) remind
   the user that keeper fills are market-executed within their slippage
   (`slippage_bps`, default 100 = 1%).

## Workflow: DCA

`place_dca_order` sells `total_amount_in` in equal chunks — `executions`
chunks (2–1000), one every `interval_seconds` (≥30s). One signature covers
the whole schedule; the approval must cover the TOTAL. Confirm the schedule
back to the user in plain language ("10 chunks of 100 USDG, one per day,
finishing on <date>") before placing.

## Workflow: review & manage

- "How are my positions?" → `get_portfolio` (balances + USD values) and
  `list_orders` (open/filled/cancelled with fill details).
- "Cancel it" → `cancel_order` with the orderHash; needs the same wallet
  that placed the order. Cancellation is signed but gasless.
- Scheduled check (daily routine / scheduled task): call `get_portfolio` +
  `list_orders`, compare with the previous run if that context is
  available, and report only meaningful changes: fills since last check,
  orders expiring within 24h, position moves the user asked to watch.
  End with "no action needed" when that's true — don't invent trades.

## Failure modes

- `KEY_MISSING` / `KEY_INVALID`: the `EPSILON_API_KEY` env is absent or
  revoked — get a free key at
  https://developers.epsilon.exchange/dashboard.
- "insufficient balance" / "allowance too low": the error names the exact
  shortfall; fix with funding or `approve_token`, don't retry blind.
- 429 with `X-RateLimit-Reset`: wait it out; never hammer quotes.
- Wallet not configured (`EPSILON_WALLET_KEY` unset): read tools still
  work; trading tools will say so. Recommend a dedicated trading wallet
  with limited funds — never the user's main wallet.

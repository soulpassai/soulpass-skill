# DeFi & Trading Cookbook

Operational wisdom for DeFi on Solana via SoulPass CLI. For command syntax and parameters, run `soulpass --help` or `soulpass schema`.

## Table of Contents

1. [Swap Strategy](#swap-strategy)
2. [Reading Lend Positions](#reading-lend-positions)
3. [Token Risk Assessment](#token-risk-assessment)
4. [Recipes — Common DeFi Patterns](#recipes)
5. [High-Frequency Daemon](#high-frequency-daemon)
6. [Pitfalls & Safety](#pitfalls)

---

## Swap Strategy

### Slippage — When to Set What

Default slippage is **50 BPS (0.5%)**. Maximum: **500 BPS (5%)**.

| Pair type | Recommended slippage | Why |
|-----------|---------------------|-----|
| Stablecoin ↔ Stablecoin (USDC/USDT) | 10 BPS (0.1%) | Deep liquidity, minimal price movement |
| Major pairs (SOL/USDC) | 50 BPS (default) | Usually fine, but set explicit during volatility |
| Low-liquidity / new tokens | 100-300 BPS | Thin orderbooks cause larger price impact |
| Meme tokens / micro-caps | 300-500 BPS | Extreme volatility, pool depth varies wildly |

If a swap fails with slippage errors, increase incrementally — don't jump straight to 500.

### Mint Address Mode — Watch the Decimals

When using mint addresses instead of symbols, `--amount` switches to **atomic units** (lamports), not human-readable:

```bash
# Symbol mode: human-readable (1.5 SOL)
soulpass swap --from SOL --to USDC --amount 1.5

# Mint mode: atomic units (1.5 SOL = 1,500,000,000 lamports)
soulpass swap --from So11...112 --to EPjF...t1v --amount 1500000000
```

This is the #1 source of "swapped way too much/too little" errors. Always use symbols when available.

---

## Reading Lend Positions

`soulpass lend balance` returns fields that are easy to misread:

```json
{
  "fTokenBalance": "100500000",
  "estimatedValue": "101200000",
  "exchangeRate": 1.006965
}
```

- **fTokenBalance** — pool share tokens you hold (NOT the underlying amount)
- **estimatedValue** — actual underlying token amount in atomic units. Divide by `10^decimals` for human-readable (USDC: divide by 1,000,000)
- **exchangeRate** — if >1.0, yield has accrued. The difference from 1.0 is your earnings percentage

**Common mistake:** checking `soulpass balance` and thinking you can withdraw from lend. Lend balance and wallet balance are separate pools. Always check `soulpass lend balance` before withdrawing.

Note: `--amount` and `--all` are mutually exclusive on `lend withdraw` — use one or the other.

---

## Token Risk Assessment

Before interacting with unknown tokens, use `soulpass price` to check risk signals:

```bash
soulpass price <token-or-mint>
```

Look at these fields in the response:
- **verified: false** — token is not verified by Jupiter. Proceed with extreme caution
- **liquidity** — low liquidity means high slippage and potential rug risk
- **marketCap** — very low market cap tokens are higher risk

Rule of thumb: if `verified: false` AND `liquidity < 100000`, think twice before swapping into it.

---

## Recipes

These are common DeFi patterns you'll encounter. Each is a goal-oriented workflow, not a command reference.

### DCA (Dollar-Cost Averaging)

Goal: Buy SOL regularly regardless of price, reducing timing risk.

```
1. soulpass price SOL                              # check current price (optional, for logging)
2. soulpass swap --from USDC --to SOL --amount 50  # buy $50 worth
3. Repeat on a schedule (hourly, daily, weekly)
```

For automated DCA, use the daemon in a loop. The agent decides the interval and amount — there's no built-in scheduler, so the agent or a cron-style tool drives the timing.

### Take-Profit / Stop-Loss

Goal: Sell when price hits a target, protect against drops.

```
1. soulpass price SOL                              # check price
2. If price >= target: soulpass swap --from SOL --to USDC --amount <sell-amount>
3. If price <= stop-loss: soulpass swap --from SOL --to USDC --amount <all>
4. Otherwise: wait and check again
```

The agent implements the logic; soulpass provides price checks and execution. Use the daemon for frequent polling to avoid startup overhead.

### Yield Parking

Goal: Earn yield on idle funds, pull out when you need to trade.

```
1. soulpass balance --token USDC                   # check idle funds
2. soulpass lend deposit --amount 100 --token USDC # park in Jupiter Lend
3. ... time passes, yield accrues ...
4. soulpass lend balance --token USDC              # check position + earnings
5. soulpass lend withdraw --token USDC --all       # pull out when ready
6. soulpass swap --from USDC --to SOL --amount ... # trade with withdrawn funds
```

Key: always check `lend balance` (not `balance`) before withdrawing. They are separate pools.

### Portfolio Rebalance

Goal: Maintain a target allocation (e.g., 60% SOL / 40% USDC).

```
1. soulpass balance                                # SOL balance
2. soulpass balance --token USDC --usd             # USDC balance in USD
3. Calculate current ratio vs target
4. If over-allocated to SOL: soulpass swap --from SOL --to USDC --amount <delta>
5. If over-allocated to USDC: soulpass swap --from USDC --to SOL --amount <delta>
```

### Monitor and Act (Signal-Driven Trading)

Goal: React to signals from another tool (Birdeye, DexScreener, custom logic).

```
1. Other tool provides signal (e.g., "new token listed", "price crossed MA")
2. soulpass price <token>                          # verify signal, check risk
3. If verified: false OR liquidity < 100000 → skip (see Token Risk Assessment)
4. soulpass swap --from USDC --to <token> --amount <size> --slippage 300
5. soulpass tx --hash <sig>                        # verify execution
```

For high-frequency signals, use the daemon: `soulpass serve` → POST swaps via JSON-RPC.

---

## High-Frequency Daemon

For trading bots, batch payments, or anything calling soulpass in a loop, use the daemon to eliminate ~600ms startup overhead per call.

```bash
soulpass serve    # starts on port 8402
```

All methods use JSON-RPC format: `POST http://127.0.0.1:8402` with `{"jsonrpc":"2.0","method":"...","params":{...},"id":1}`.

**Available methods:** `price`, `balance`, `tx_status`, `swap`, `pay`, `batch` (up to 20 parallel SOL transfers), `cache_invalidate`.

### When to Use Daemon vs CLI

| Scenario | Use |
|----------|-----|
| One-off payment or swap | CLI (`soulpass pay/swap ...`) |
| 10+ operations in a loop | Daemon |
| Trading bot polling prices every few seconds | Daemon |
| Interactive agent session with occasional operations | CLI (simpler) |

The daemon keeps warm caches — after the first `price` call, subsequent calls are significantly faster.

---

## Pitfalls

### Gas: Authority Pays, Not Wallet

After 30 free sponsored transactions per month, gas comes from your **Authority address**, not your Wallet/Vault. If Authority runs out of SOL, all transactions fail — even if Wallet has plenty of SOL.

Check: `soulpass balance --address <authority-address>`

### Verify After Every Swap

Always check the tx hash after submission: `soulpass tx --hash <sig>`. Don't assume success from the CLI response alone — network congestion can cause transactions to land late or fail.

### Don't Confuse the Three Balance Pools

This is repeated from SKILL.md because it's that important:

```
soulpass balance           → Wallet (Vault) — your funds
soulpass lend balance      → Lend position — deposited yield funds
Authority address          → Gas — keeps transactions flowing
```

Checking the wrong pool before an operation is the #1 source of "insufficient balance" errors.

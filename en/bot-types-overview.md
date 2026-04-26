# Bot Types Overview + Selection Guide

> **For**: Operators deciding which bot fits their situation.
> **TL;DR**: Three specialized bots — different goals demand different bots.

---

## 1. At a Glance

| Bot | One-line goal | Main revenue | Best for |
|---|---|---|---|
| **Volume-MM** | Synthetic volume + price climb | Fee incentives / market activity | Token issuers, listing-criteria volume |
| **Role-MM** | Real market making (spread capture) | Bid/ask spread | Professional market makers |
| **Arbitrage** | Cross-exchange price gap profit | Price differential between venues | Arbitrage operators |

**Detailed operation settings live in per-bot guides** (currently only Volume-MM is documented; Role-MM/Arbitrage guides coming).

---

## 2. Volume-MM Bot

### What it does
- Self-cross between the user's own two sub-accounts (bid wallet / ask wallet) to generate volume
- Optional: gradually push price upward (Walk-Up Engine)
- Auto-defense system (Asymmetric Defense): blocks ladder when abnormal market detected

### When to use
- New token after issuance with insufficient volume — needs to clear exchange listing/relisting criteria
- Maintain a specific price band — keep the orderbook active
- Gradual price increase — natural climb in bullish flow

### Suitable markets
✅ Stable flow (daily volatility < 5%)  
✅ Balanced buy/sell (24h flow score -0.3 ~ +0.3)  
✅ Some external trader presence (≥ 10 trades/h)

### Unsuitable markets
❌ Strong sell pressure (24h flow score < -0.5) — climb attempts bleed USDT
❌ Thick sell wall + thin bid liquidity — one-sided absorption risk
❌ Extremely low liquidity (< 5 trades/h) — fills barely happen

### Capital guide
- Minimum: USDT $300 + base asset (size must clear exchange minNotional)
- Recommended: USDT $600+ across 2-3 wallets — gives the asymmetric defense room to operate

### Operating modes (choose one)
1. **Self-cross only (safety first)**: walkUp disabled, generate volume without moving price — [safe-recipe.md](./safe-recipe.md)
2. **Walk-Up climb (recovery markets)**: walkUp.enabled=true, target price moves up gradually — analyze first ([market-analysis.md](./market-analysis.md))

---

## 3. Role-MM Bot

### What it does
- Three sub-accounts: bid (USDT side) / ask (coin side) / settle (rebalancing)
- **Posts limit orders on both sides** — captures spread when one side fills
- Auto-tunes spread based on volatility (Avellaneda-Stoikov)
- Auto-rebalances via settle account when one side is depleted

### When to use
- Stable market making for **continuous spread revenue**
- Provide natural orderbook depth for a token (both sides)
- Hedge integration — spot MM on one venue + hedge on another (XEMM)
- Unwind large positions slowly without market impact (TWAP)

### Suitable markets
✅ Mid-tier tokens (decent volume, stable spread)  
✅ Low to medium volatility (Avellaneda model is most effective here)  
✅ Exchanges that support sub-account auto-transfer (Binance / OKX / KuCoin etc.)

### Unsuitable markets
❌ Highly volatile markets (Avellaneda's dynamic spread can go negative-margin)
❌ Extremely low liquidity (frequent inventory depletion)
❌ Very high fee exchanges (spread < fee → losses)

### Capital guide
- Minimum: USDT $500 × 3 wallets (bid + ask + settle)
- Recommended: USDT $2000+ × 3 — absorbs inventory swings + sustained operation
- Coin holdings: average daily volume × 1-2 days

### Difference from Volume-MM
- Volume-MM: self-cross between own wallets → generates volume, no profit
- Role-MM: trades with real external counterparties → spread profit, real market formation

---

## 4. Arbitrage Bot

### What it does
- Subscribes to orderbooks of two exchanges (e.g. MEXC + Binance) in real time
- When price gap exceeds fees + threshold, executes:
  - Buy on cheaper venue + sell on pricier venue (simultaneous orders)
- Manages partial-fill risk (retries + imbalance guard)

### When to use
- Right after a new token listing — large gaps between venues
- Idle capital efficiency — theoretically risk-free (when both legs fill)
- Auto-balance capital between two exchanges

### Suitable markets
✅ Newly listed tokens (price discovery phase, 0.5~5% gaps possible)  
✅ Low to mid liquidity pairs (major coins have basically zero arb)  
✅ Two exchanges with stable latency

### Unsuitable markets
❌ Major coins (BTC, ETH) — gaps < 0.05%, won't cover fees
❌ High network latency — only one leg fills, position risk
❌ Fees > 0.5% per side — net loss

### Capital guide
- Minimum: USDT $1000 × 2 exchanges
- Recommended: USDT $5000+ × 2 — captures bigger opportunities + absorbs partial fills

### Risk factors
- **Partial fill**: only one leg executes → one-sided position → loss on price reversal
- **Exchange latency**: stale WS data → chase phantom opportunity → loss
- **Fees**: total fees on both sides exceeding the gap → net negative

---

## 5. Decision Matrix

| Situation / Goal | Recommended Bot |
|---|---|
| New token, need listing-criteria volume | **Volume-MM** (self-cross only) |
| New token, price discovery + cross-venue gaps | **Arbitrage** |
| Stable token, continuous spread revenue | **Role-MM** |
| Gradual token price climb + volume at the same time | **Volume-MM** (Walk-Up + market analysis) |
| Unwind large position slowly (no impact) | **Role-MM** (TWAP executor) |
| Hedge across venues (spot+futures or two exchanges) | **Role-MM** (XEMM) |
| Capital < $1000 | **Volume-MM** (most capital-efficient) |

---

## 6. Universal Safety Principles (All Bots)

1. **Analyze the market first** — measure the 4 indicators in [market-analysis.md](./market-analysis.md)
2. **Conservative capital first** — start small, verify stability, then scale
3. **Continuous monitoring** — hourly checks for the first 6h
4. **Set alarm thresholds** — recommend stop at USDT -2% from start (conservative)
5. **Trust the defense layer** — Circuit Breakers / Defense Mode auto-activate

---

## 7. Detailed Operations

- [operation-guide.md](./operation-guide.md) — General operation manual
- [safe-recipe.md](./safe-recipe.md) — Volume-MM safe setting (verified recipe)
- [market-analysis.md](./market-analysis.md) — 4-indicator market flow analysis
- Role-MM / Arbitrage guides coming later — for now, refer to the dashboard's bot creation form + tooltips

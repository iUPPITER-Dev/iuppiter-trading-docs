# Market Flow 4-Indicator Analysis — Bot Decision Guide

> **When to use**: Before starting a new bot / before enabling climb mode / before changing bot settings.
> **Goal**: Objectively judge whether market flow favors our strategy.

---

## 1. Why Analyze

Most operational failures occur when "the market is unfavorable but the setting is aggressive":
- Climb attempt during external buyer absence → USDT bleed guaranteed
- Frontal attack on thick sell wall → sellers absorb our USDT
- Forcing activity in a low-frequency market → fees accumulate as net loss

**Looking at all 4 indicators together** instantly shows the market's friendliness / risk level.

---

## 2. The 4 Indicators

### 2.1 24h Flow Score
**Definition**: `(buy_taker_usdt - sell_taker_usdt) / total`  
**Interpretation**: External market buy/sell balance. Positive = buy pressure, negative = sell pressure.

| Value | Meaning | Bot strategy |
|---|---|---|
| `> +0.3` | Buy pressure (bull) | Climb mode possible |
| `+0.3 ~ -0.3` | Balanced (sideways) | Self-cross + light climb |
| `-0.3 ~ -0.5` | Sell pressure (bear) | Self-cross only |
| `< -0.5` | Strong sell (collapse) | Self-cross with caution, USDT preservation #1 |

### 2.2 Sell Wall Depth
**Definition**: Cumulative ask USDT from current mid up to target price.  
**Interpretation**: Volume we'd have to push through during climb.

| Comparison | Meaning |
|---|---|
| Wall USDT < our USDT × 50% | Passable (low climb cost) |
| Wall USDT ≈ our USDT | Risky (capital nearly depleted) |
| Wall USDT > our USDT × 200% | Climb infeasible (cannot frontally pass) |

### 2.3 Bid Liquidity Depth
**Definition**: Cumulative bid USDT in a band below current mid (e.g. -5% to -10%).  
**Interpretation**: Buy support that catches mid when external sellers push it down.

| Comparison | Meaning |
|---|---|
| > $500 | Strong safety net |
| $100~500 | Moderate (our bot ladder may need to provide support) |
| < $100 | Very thin — price free-fall risk on selling |

### 2.4 24h Trade Frequency
**Definition**: Number of external trades in 24h ÷ hours.  
**Interpretation**: Market liveness. Too low = our ladder rarely gets hit.

| Frequency | Meaning |
|---|---|
| > 50/h | Active (self-cross + external hits frequent) |
| 10~50/h | Moderate |
| < 10/h | Stagnant (our bot dominates market volume) |

---

## 3. Measurement (Node Script Examples)

### 3.1 Orderbook Depth (Sell Wall + Bid Liquidity)

```javascript
const r = await fetch('https://api.mexc.com/api/v3/depth?symbol=IUPUSDT&limit=5000')
const ob = await r.json()
const bid = parseFloat(ob.bids[0][0]), ask = parseFloat(ob.asks[0][0])
const mid = (bid + ask) / 2

// Sell wall: cumulative up to $0.005
let askUsdt = 0
for (const [p, q] of ob.asks) {
  const price = parseFloat(p), qty = parseFloat(q)
  if (price <= 0.005) askUsdt += price * qty
}

// Bid liquidity: cumulative below mid
let bidUsdt = 0
for (const [p, q] of ob.bids) {
  bidUsdt += parseFloat(p) * parseFloat(q)
}
console.log(`mid=${mid} askWall=$${askUsdt.toFixed(0)} bidDepth=$${bidUsdt.toFixed(0)}`)
```

### 3.2 24h Flow Score + Trade Frequency

```javascript
const tEnd = Date.now()
const tStart = tEnd - 24 * 60 * 60_000
let buyUsdt = 0, sellUsdt = 0, count = 0
let cursor = tStart
while (cursor < tEnd) {
  const r = await fetch(`https://api.mexc.com/api/v3/aggTrades?symbol=IUPUSDT&startTime=${cursor}&endTime=${cursor + 60*60_000}&limit=1000`)
  const trades = await r.json()
  if (Array.isArray(trades)) for (const t of trades) {
    const usdt = parseFloat(t.p) * parseFloat(t.q)
    if (t.m) sellUsdt += usdt; else buyUsdt += usdt
    count++
  }
  cursor += 60 * 60_000
}
const flow = (buyUsdt - sellUsdt) / (buyUsdt + sellUsdt)
console.log(`24h: buy=$${buyUsdt.toFixed(0)} sell=$${sellUsdt.toFixed(0)} flow=${flow.toFixed(3)} count=${count} (${(count/24).toFixed(1)}/h)`)
```

> MEXC `t.m` field: `true` = buyer is maker (= seller is taker, sell-taker), `false` = seller is maker (= buyer is taker, buy-taker).

---

## 4. Decision Matrix

| flow | Sell wall | Bid depth | Trade freq | **Recommended setting** |
|---|---|---|---|---|
| > 0 | < $300 | > $200 | > 30/h | **Climb mode** (Walk-Up enabled) |
| > 0 | < $500 | > $100 | > 10/h | **Light climb** (conservative ETA) |
| -0.3 ~ +0.3 | any | any | any | **Self-cross + light climb** |
| < -0.3 | any | any | any | **Self-cross only** ([safe-recipe](./safe-recipe.md)) |
| < -0.5 | any | any | any | **Self-cross only + tickInterval 240s** (half risk) |
| any | any | < $100 | < 10/h | **Bot pause recommended** (market nearly dead) |

---

## 5. Case Study: 2026-04-25 IUPUSDT

**Measurements (before canary bot start):**
- 24h flow score: **-0.81** (buy $87 / sell $853, strong sell pressure)
- Sell wall (up to $0.005): **$2,792** (impossible to pass with our $640 USDT)
- Bid liquidity (below mid): **$138** (very thin)
- Trade frequency: **16.6/h** (low)

**Decision**: Per the table above, `flow < -0.5` → **Self-cross only + tickInterval 240s** applied.

**Result (11h operation)**:
- USDT $642 → $658.88 (**+$16.88, +2.63%**)
- Absorbed -4.89% mid swing
- Market quoteVolume $1,253 → $1,441 (activity ↑)
- Bot absorbed market volatility while preserving USDT and maintaining volume

→ Textbook case of analysis-driven conservative setting → stable operation.

---

## 6. Analysis Automation (Optional)

Hourly auto-measurement of the 4 indicators + Slack/Telegram alerts:
- Alert when flow score crosses < -0.5
- Alert when sell wall changes ±20%
- Alert when trade frequency stays < 10/h

→ Faster reaction to market changes, less operator decision burden.

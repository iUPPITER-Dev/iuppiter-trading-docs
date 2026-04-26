# Volume-MM Bot Operation Guide

> **Audience**: Operators using the IUPPITER Volume-MM bot to generate trading volume or adjust price.
> **Prerequisites**: Familiarity with exchange API keys, orderbook basics, and self-cross/wash-trade concepts.

---

## 1. What is the Volume-MM Bot

The Volume-MM bot combines **self-cross trading** and **optional Walk-Up (price climb)** to produce volume and market activity:

- **Self-cross**: Places best-pair matched orders between two wallets owned by the same user (bid wallet / ask wallet) → both wallets trade with each other → recorded as MEXC volume
- **Walk-Up**: Mechanism that gradually moves the baseline toward a target price to shift mid (can be enabled/disabled)

A 5-layer **defense system** (Asymmetric Defense) runs in the background:
- Detects abnormal price moves, inventory bias, external sell pressure, and other risks → automatically pauses the bot or makes the ladder more conservative.

---

## 2. Pre-Start Checklist

### 2.1 Capital Setup
| Wallet | Role | Recommended balance |
|---|---|---|
| `T-M1` (bid) | Buy-side ladder | USDT $300+ + some IUP |
| `T-M2` (ask) | Sell-side ladder | USDT $300+ + some IUP |
| `T-L` (settle) | Settlement (optional) | small IUP |

### 2.2 Market Flow Analysis (Required)
**Before starting any bot**, measure the 4 indicators in [market-analysis.md](./market-analysis.md):
1. 24h flow score (buy/sell taker ratio)
2. Sell wall depth (cumulative ask USDT up to target price)
3. Bid liquidity depth
4. 24h trade frequency

→ If flow score < -0.5 (strong sell pressure), **do not attempt climb mode**. Use the self-cross only mode in [safe-recipe.md](./safe-recipe.md).

### 2.3 minNotional Check
MEXC's IUPUSDT minimum order size is **$1 USDT**. Make sure `order size × current mid ≥ $1`.
Example: mid $0.004 → minimum order size 250 IUP, recommended 350 IUP for safety (= $1.4).

---

## 3. Bot Creation

In the dashboard (`https://mm.iuppiter.io/bots`), click "New Bot" and enter:

| Field | Recommended |
|---|---|
| Symbol | `IUPUSDT` |
| Capital | USDT $640+ |
| API keys | T-M1 (bid), T-M2 (ask), T-L (settle) |
| Target price | Near current mid (in unfavorable climb conditions) |
| Tolerance | 5% |
| Walk-Up | Only enable when buy pressure is strong |
| Defense | bias=balanced, enforcement=enforced |
| Daily volume | $200~500 USDT |

**Detailed safe setting**: [safe-recipe.md](./safe-recipe.md)

---

## 4. Live Monitoring

### 4.1 Dashboard
- `mode`: NORMAL / ENGAGED_BUY / ENGAGED_SELL / DEFENSIVE / CRASH / RECOVERY
- `walkUpState`: INIT / CLIMBING / MAINTAIN / HOLDING / RETREATING / STAGED_CLIMB / PULSE
- `breakers`: 5 types (NetTaker / PriceMove / StopLoss / InventorySkew / BookSanity)

### 4.2 Key Indicators
| Metric | Healthy | Warning | Stop immediately |
|---|---|---|---|
| mode | NORMAL | DEFENSIVE briefly | CRASH sustained |
| breaker trips | 0 | occasional | 5+ in 1h |
| USDT change | profit / sideways | -$2 from start | -$5 from start |
| fill rate | 0.5~2/min | <0.3/min | 0 (5min+) |

### 4.3 Recommended Cadence
- **First 1h**: Close monitoring every 5-10 min
- **1h ~ 6h**: Check dashboard hourly
- **6h+**: Check every 6-12h or only on alarm

### 4.4 Prod Log Inspection (When Needed)
If dashboard metrics show 0 but balances/orders look fine, see the prod log grep pattern in [troubleshooting](#).

---

## 5. Stop + Capital Recovery

### 5.1 Normal Stop
- Click "Stop Bot" in the dashboard → backend executes `cancelAllOrders` on every wallet
- Or via API: `POST /api/v1/bots/{botId}/stop`

### 5.2 Post-Stop Verification
- All wallets show 0 open orders
- Locked balance = 0 (everything moved to free)
- IUP/USDT balance shifted reasonably from start (self-cross fees + spread extraction ± mid movement)

### 5.3 Capital Recovery (If Needed)
T-L → main wallet via `internalTransfer` or external withdrawal. If IUP is split across wallets, consolidate first then withdraw.

---

## 6. Alarm Response Manual

| Alarm | Immediate action |
|---|---|
| 1 breaker trip | Observe 1h, confirm auto-recovery |
| 5+ breakers in 1h | Increase tickInterval to 240s, observe 1h more |
| USDT -$5 from start | Stop immediately, re-analyze market and adjust setting |
| mode CRASH 5min+ | Stop immediately, check prod log + external market |
| fill_count = 0 for 1h | Inspect prod log (grep `bestPairFilled`), possible wiring issue |
| External sell wave (mid -10%+) | Recommend stop, restart after stability returns |

---

## 7. Further Reading

- [safe-recipe.md](./safe-recipe.md) — Self-cross only safe setting (verified recipe)
- [market-analysis.md](./market-analysis.md) — Market flow 4-indicator analysis
- [incident-2026-04-24.md](./incident-2026-04-24.md) — Incident retrospective + asymmetric defense narrative

# Self-Cross Only Safe Operating Recipe

> **Verified setting**: Achieves USDT preservation + volume maintenance simultaneously through conservative operation.
> **Verification window**: 2026-04-25 ~ 04-26 canary, 11h (USDT $642 → $658.88, +$16.88 preserved).

---

## 1. When to Use This Recipe

Apply this recipe **at startup** if any of the following holds:
- 24h flow score < -0.5 (no external buyers)
- Sell wall too thick (cannot pass with available capital)
- High volatility, unclear direction
- New bot — want to verify stability first

→ This recipe does **not** attempt to push price up. It only maintains volume and market activity through self-cross.

---

## 2. Setting Detail (DB `volume_mm_config`)

```json
{
  "walkUp": { "enabled": false },
  "targetPrice": { "price": <current mid>, "tolerancePct": 5 },
  "defense": {
    "bias": "balanced",
    "enforcement": "enforced",
    "thresholdsOverride": { "inventorySkewRatio": 1 }
  },
  "engineConfig": {
    "symbol": "IUPUSDT",
    "tickIntervalMs": 240000,
    "bestPairSize": 350,
    "ladder": {
      "layers": 5,
      "layerShape": "uniform",
      "baseOrderSize": 350,
      "baseSpreadPct": 0.003,
      "spreadIncrementPct": 0.0015,
      "sizeJitterPct": 0.1,
      "priceJitterTicks": 1,
      "placementDelayMsMin": 100,
      "placementDelayMsMax": 500,
      "maxBudgetUsdtPerTick": 30
    }
  },
  "dailyVolumeUsdt": 200
}
```

### Key Parameter Rationale

| Parameter | Value | Reason |
|---|---|---|
| `walkUp.enabled` | `false` | Don't attempt climb — prevents USDT bleed |
| `targetPrice.price` | current mid | Ladder forms tightly around mid |
| `tolerancePct` | 5 | Activity confined to mid ±5% |
| `tickIntervalMs` | 240000 (240s) | Half-frequency cycle → half fee cost → half risk |
| `bestPairSize` / `baseOrderSize` | 350 | At mid $0.0036 → $1.26 (clears $1 minNotional) |
| `baseSpreadPct` | 0.003 (0.3%) | Sufficient bid/ask separation |
| `inventorySkewRatio` | 1 | Prevents instant trip from initial IUP-skewed balance |

---

## 3. Mechanism (Verified by 11h Canary Observation)

The bot self-balances based on market direction:

- **Mid drops** (-1~-5%):
  - External sellers hit our bid-side ladder (above mid)
  - We: buy IUP, spend USDT temporarily (-)
  - Example: at 9h mark, -$3.15 short-term loss

- **Mid recovers** (+1~+2.5%):
  - External buyers hit our ask-side ladder (below mid)
  - We: sell IUP, recoup USDT (+)
  - Example: at 10h mark, +$3.32 recovery

- **Mid sideways** (±0.5%):
  - Only self-cross runs, slight fee deduction
  - USDT change ~0 (neutral)

→ In sideways/light-volatility markets, **self-cross fees + spread extraction net positive**.

---

## 4. Canary 11h Verification Results

| Time | Mid change | USDT (vs start $642) |
|---|---|---|
| 1h | stable | **+$18.10** (absorbed external buying) |
| 3h (peak) | +0.08% | **+$19.52** ⬆ |
| 6h | -4.89% cumulative | +$6.28 |
| 8h | (240s tick applied) | +$6.25 |
| 9h 🚨 | -1.45% | **-$3.15** (short-term alarm) |
| 10h | +2.55% recovery | +$3.32 |
| 11h | -5.76% volatile | +$1.27 |
| **stop** | (mid recovered) | **+$16.88 final** ✅ |

**Over 6h: NORMAL 100%, breaker trips 0, absorbed market volatility, capital recovered.**

---

## 5. Alarm Thresholds (For This Recipe)

| Alarm | Threshold | Action |
|---|---|---|
| **L1 caution** | USDT -$3 from start | Observe 1h, check market flow |
| **L2 warning** | USDT -$5 from start | Recommend immediate stop (pre-agreed stop loss) |
| **L3 immediate stop** | mode CRASH 5min+ or mid -10%+ | Stop unconditionally |

---

## 6. Variant Options (When Needed)

| Situation | Adjustment |
|---|---|
| Want more volume | `tickIntervalMs` 240s → 120s (2× cycle, 2× risk) |
| Want lower risk | `tickIntervalMs` 240s → 360s, or `baseSpreadPct` 0.003 → 0.005 (wider spread) |
| Sell wall too close | `tolerancePct` 5 → 3 (narrower band) |
| Buy pressure emerges | Set walkUp.enabled=true + targetPrice slightly above mid (start climb) |

---

## 7. Next Steps

- Confirm stability (6h+) → expand to other coins / capital tiers
- Buy pressure detected → consider climb mode ([market-analysis.md](./market-analysis.md))
- On incident → see [incident-2026-04-24.md](./incident-2026-04-24.md)

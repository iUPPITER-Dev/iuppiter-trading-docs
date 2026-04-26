# Self-Cross Only Safe Operating Recipe

> **Verified setting**: Achieves USDT preservation + volume maintenance simultaneously through conservative operation. Stability confirmed through internal verification.

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

> Set `bestPairSize` / `baseOrderSize` so that `size × current mid` clears the exchange minNotional ($1 USDT).

### Key Parameter Rationale

| Parameter | Value | Reason |
|---|---|---|
| `walkUp.enabled` | `false` | Don't attempt climb — prevents USDT bleed |
| `targetPrice.price` | current mid | Ladder forms tightly around mid |
| `tolerancePct` | 5 | Activity confined to mid ±5% |
| `tickIntervalMs` | 240000 (240s) | Half-frequency cycle → half fee cost → half risk |
| `baseSpreadPct` | 0.003 (0.3%) | Sufficient bid/ask separation |
| `inventorySkewRatio` | 1 | Prevents instant trip from initial asset-skewed balance |

---

## 3. Mechanism

The bot self-balances based on market direction:

- **Mid drops**:
  - External sellers hit our bid-side ladder (above mid)
  - We: buy base asset, spend USDT temporarily (-)

- **Mid recovers**:
  - External buyers hit our ask-side ladder (below mid)
  - We: sell base asset, recoup USDT (+)

- **Mid sideways**:
  - Only self-cross runs, slight fee deduction
  - USDT change ~0 (neutral)

→ In sideways/light-volatility markets, **self-cross fees + spread extraction can net positive**.

---

## 4. Alarm Thresholds (For This Recipe)

| Alarm | Threshold | Action |
|---|---|---|
| **L1 caution** | USDT -$3 from start | Observe 1h, check market flow |
| **L2 warning** | USDT -$5 from start | Recommend immediate stop (pre-agreed stop loss) |
| **L3 immediate stop** | mode CRASH 5min+ or mid -10%+ | Stop unconditionally |

---

## 5. Variant Options (When Needed)

| Situation | Adjustment |
|---|---|
| Want more volume | `tickIntervalMs` 240s → 120s (2× cycle, 2× risk) |
| Want lower risk | `tickIntervalMs` 240s → 360s, or `baseSpreadPct` 0.003 → 0.005 (wider spread) |
| Sell wall too close | `tolerancePct` 5 → 3 (narrower band) |
| Buy pressure emerges | Set walkUp.enabled=true + targetPrice slightly above mid (start climb) |

---

## 6. Next Steps

- Confirm stability (6h+) → expand to other coins / capital tiers
- Buy pressure detected → consider climb mode ([market-analysis.md](./market-analysis.md))
- General operations → [operation-guide.md](./operation-guide.md)

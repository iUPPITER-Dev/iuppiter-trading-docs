# 自己クロス only 安全運用レシピ

> **検証済み setting**: USDT 保全 + 取引量維持を同時達成する保守的な運用。
> **検証期間**: 2026-04-25 ~ 04-26 canary 11h (USDT $642 → $658.88、+$16.88 保全)。

---

## 1. このレシピを使う場面

以下のいずれかに該当すれば **このレシピで開始**:
- 24h flow score < -0.5 (外部買い手不在)
- 売り壁が厚い (capital で通過不可)
- 市場のボラティリティが高く方向不明
- 新規ボット起動、安定性検証から始めたい時

→ このレシピは価格上昇を試行**しません**。自己クロスで取引量 + 市場活性度のみ維持。

---

## 2. Setting 詳細 (DB `volume_mm_config`)

```json
{
  "walkUp": { "enabled": false },
  "targetPrice": { "price": <現在の mid>, "tolerancePct": 5 },
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

### 主要パラメータ説明

| パラメータ | 値 | 理由 |
|---|---|---|
| `walkUp.enabled` | `false` | climb 試行なし — USDT 流出防止 |
| `targetPrice.price` | 現在の mid | ladder が mid の両側に狭く形成 |
| `tolerancePct` | 5 | mid ±5% 範囲内のみで活動 |
| `tickIntervalMs` | 240000 (240s) | cycle 頻度を半分に → fee コスト半減 → リスク半減 |
| `bestPairSize` / `baseOrderSize` | 350 | mid $0.0036 で $1.26 (minNotional $1 通過) |
| `baseSpreadPct` | 0.003 (0.3%) | bid/ask 十分に分離 |
| `inventorySkewRatio` | 1 | 開始時の IUP 偏り残高による即時 trip 防止 |

---

## 3. メカニズム (canary 11h 観測検証)

ボットは市場の流れに応じて自動的にバランス:

- **mid 下落フロー** (-1~-5%):
  - 外部の売り手が私たちの ladder の bid 側 (mid 上) を hit
  - 私たち: IUP 購入、USDT 一時的に支出 (短期 -)
  - 例: 9h 時点で -$3.15 短期損失

- **mid 回復フロー** (+1~+2.5%):
  - 外部の買い手が私たちの ladder の ask 側 (mid 下) を hit
  - 私たち: IUP 売却、USDT 回収 (+)
  - 例: 10h 時点で +$3.32 回復

- **mid 横ばい** (±0.5%):
  - self-cross のみ稼働、自己クロス fee がわずかに差し引かれる
  - USDT 変化ほぼ 0 (中立)

→ 市場が横ばい/弱変動の環境では、**自己クロス fee + spread 抽出で net positive**。

---

## 4. canary 11h 検証結果

| 時点 | mid 変化 | USDT (開始 $642 比) |
|---|---|---|
| 1h | 安定 | **+$18.10** (外部買い吸収) |
| 3h (peak) | +0.08% | **+$19.52** ⬆ |
| 6h | -4.89% 累積 | +$6.28 |
| 8h | (240s tick 適用) | +$6.25 |
| 9h 🚨 | -1.45% | **-$3.15** (短期 alarm) |
| 10h | +2.55% 回復 | +$3.32 |
| 11h | -5.76% 変動 | +$1.27 |
| **stop** | (mid 回復) | **+$16.88 最終** ✅ |

**6 時間中 NORMAL 100%、breaker trip 0、市場変動を吸収しつつ資本回収完了。**

---

## 5. アラーム基準 (このレシピ適用時)

| アラーム | 基準 | 対応 |
|---|---|---|
| **L1 注意** | USDT -$3 from 開始 | 1h 観測、市場フロー確認 |
| **L2 警告** | USDT -$5 from 開始 | 即停止推奨 (事前合意の stop loss) |
| **L3 即停止** | mode CRASH 5min+ または mid -10%+ | 無条件停止 |

---

## 6. 変形オプション (必要時)

| 状況 | 調整 |
|---|---|
| 取引量を増やしたい | `tickIntervalMs` 240s → 120s (cycle 2 倍、リスク 2 倍) |
| リスクをさらに減らしたい | `tickIntervalMs` 240s → 360s または `baseSpreadPct` 0.003 → 0.005 (spread 拡大) |
| 売り壁が近い | `tolerancePct` 5 → 3 (band をさらに狭く) |
| 市場の買い登場 | walkUp.enabled=true + targetPrice を少し上に (climb 開始) |

---

## 7. 次のステップ

- ボット安定運用確認 (6h+) → 他の coin / capital tier に拡張
- 市場の買い登場 → climb モード検討 ([market-analysis.md](./market-analysis.md))
- 事故発生 → [incident-2026-04-24.md](./incident-2026-04-24.md) 参照

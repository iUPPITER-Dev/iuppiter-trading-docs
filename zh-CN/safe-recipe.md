# 自交易 only 安全运营配方

> **验证过的 setting**: 同时实现 USDT 保护 + 交易量维持的保守运营。已通过内部验证确认稳定性。

---

## 1. 何时使用此配方

满足以下任一条件即**以此配方启动**:
- 24h flow score < -0.5 (外部买家缺失)
- 卖墙过厚 (capital 无法穿越)
- 市场波动性大且方向不明
- 新机器人启动,想先验证稳定性

→ 此配方**不**尝试推升价格。仅通过自交易维持交易量 + 市场活跃度。

---

## 2. Setting 详情 (DB `volume_mm_config`)

```json
{
  "walkUp": { "enabled": false },
  "targetPrice": { "price": <当前 mid>, "tolerancePct": 5 },
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

> 设置 `bestPairSize` / `baseOrderSize` 使其满足交易所 minNotional ($1 USDT),按当前 mid 调整。

### 主要参数说明

| 参数 | 值 | 理由 |
|---|---|---|
| `walkUp.enabled` | `false` | 不尝试 climb — 防止 USDT 流失 |
| `targetPrice.price` | 当前 mid | ladder 在 mid 两侧紧密形成 |
| `tolerancePct` | 5 | 仅在 mid ±5% 内活动 |
| `tickIntervalMs` | 240000 (240s) | cycle 频率减半 → fee 成本减半 → 风险减半 |
| `baseSpreadPct` | 0.003 (0.3%) | bid/ask 充分分离 |
| `inventorySkewRatio` | 1 | 防止初始资产偏向余额引起的即时 trip |

---

## 3. 机制

机器人根据市场流向自动平衡:

- **mid 下跌流向**:
  - 外部卖家击中我们 ladder 的 bid 侧 (mid 上方)
  - 我们: 买入基础资产,USDT 暂时支出 (短期 -)

- **mid 恢复流向**:
  - 外部买家击中我们 ladder 的 ask 侧 (mid 下方)
  - 我们: 卖出基础资产,USDT 回收 (+)

- **mid 横盘**:
  - 仅 self-cross 工作,自交易 fee 略微扣除
  - USDT 变化几乎为 0 (中性)

→ 在横盘/弱波动环境下,**自交易 fee + spread 抽取可 net positive**。

---

## 4. 警报标准 (此配方应用时)

| 警报 | 标准 | 处理 |
|---|---|---|
| **L1 注意** | USDT -$3 from 起始 | 1h 观察,检查市场流向 |
| **L2 警告** | USDT -$5 from 起始 | 推荐立即停止 (事先约定的 stop loss) |
| **L3 立即停止** | mode CRASH 5min+ 或 mid -10%+ | 无条件停止 |

---

## 5. 变形选项 (必要时)

| 状况 | 调整 |
|---|---|
| 想增加交易量 | `tickIntervalMs` 240s → 120s (cycle 2 倍,风险 2 倍) |
| 想进一步降低风险 | `tickIntervalMs` 240s → 360s 或 `baseSpreadPct` 0.003 → 0.005 (spread 扩大) |
| 卖墙临近 | `tolerancePct` 5 → 3 (band 进一步收窄) |
| 市场买入登场 | walkUp.enabled=true + targetPrice 略微上调 (climb 开始) |

---

## 6. 下一步

- 确认机器人稳定运营 (6h+) → 扩展到其他 coin / capital tier
- 市场买入登场 → 考虑 climb 模式 ([market-analysis.md](./market-analysis.md))
- 一般运营 → [operation-guide.md](./operation-guide.md)

# 市场流向 4 指标分析 — 机器人运营决策指南

> **使用时机**: 新机器人启动前 / climb 模式启用前 / 机器人 setting 变更前。
> **目的**: 客观判断市场流向是否对我们的策略有利。

---

## 1. 为何分析

机器人运营失败的大多数案例发生于「市场对我们不利但 setting 激进」:
- 外部买家缺失环境下尝试 climb → USDT 流失保证
- 厚卖墙正面突破 → 卖家吸收我们的 USDT
- 在低交易频率市场强行活跃 → fee 仅累积损失

**4 指标一起看**,市场的友好度 / 风险度一目了然。

---

## 2. 4 指标定义

### 2.1 24h flow score
**定义**: `(buy_taker_usdt - sell_taker_usdt) / total`  
**解释**: 外部市场的买/卖平衡。正 = 买压强,负 = 卖压强。

| 值 | 含义 | 机器人策略 |
|---|---|---|
| `> +0.3` | 买压 (强势) | climb 尝试可行 |
| `+0.3 ~ -0.3` | 平衡 (横盘) | 自交易 + 弱 climb |
| `-0.3 ~ -0.5` | 卖压 (弱势) | 自交易 only |
| `< -0.5` | 强卖压 (崩盘) | 自交易需谨慎,USDT 保护第一 |

### 2.2 卖墙 depth
**定义**: 当前 mid 至目标价格的累积 ask USDT。  
**解释**: climb 时需要穿越的外部卖出物量。

| 比较 | 含义 |
|---|---|
| 卖墙 USDT < 我们 USDT × 50% | 可穿越 (climb 成本低) |
| 卖墙 USDT ≈ 我们 USDT | 风险 (资本几近耗尽) |
| 卖墙 USDT > 我们 USDT × 200% | climb 困难 (无法正面突破) |

### 2.3 买方流动性厚度
**定义**: 当前 mid 下方一定区间 (-5%~ -10%) 的累积 bid USDT。  
**解释**: 外部抛售将 mid 推下时承接的买入物量。

| 比较 | 含义 |
|---|---|
| 买方流动性 > $500 | 安全网充足 |
| 买方流动性 $100~500 | 普通 (防御需要时机器人 ladder 起承接作用) |
| 买方流动性 < $100 | 非常薄,抛售时价格自由落体风险 |

### 2.4 24h 交易频率
**定义**: 24h 内外部 trade 数 / 小时。  
**解释**: 市场活跃度。过低则我们 ladder 被击中的频率也低。

| 频率 | 含义 |
|---|---|
| > 50/h | 活跃 (自交易 + 外部 hit 频繁) |
| 10~50/h | 普通 |
| < 10/h | 停滞 (我们机器人占市场交易量大半) |

---

## 3. 测量方法 (Node 脚本示例)

### 3.1 Orderbook depth (卖墙 + 买方流动性)

```javascript
const r = await fetch('https://api.mexc.com/api/v3/depth?symbol=IUPUSDT&limit=5000')
const ob = await r.json()
const bid = parseFloat(ob.bids[0][0]), ask = parseFloat(ob.asks[0][0])
const mid = (bid + ask) / 2

// 卖墙: 至 $0.005 的累积
let askUsdt = 0
for (const [p, q] of ob.asks) {
  const price = parseFloat(p), qty = parseFloat(q)
  if (price <= 0.005) askUsdt += price * qty
}

// 买方流动性: mid 下方累积
let bidUsdt = 0
for (const [p, q] of ob.bids) {
  bidUsdt += parseFloat(p) * parseFloat(q)
}
console.log(`mid=${mid} askWall=$${askUsdt.toFixed(0)} bidDepth=$${bidUsdt.toFixed(0)}`)
```

### 3.2 24h flow score + 交易频率

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

> MEXC `t.m` 字段: `true` = 买家是 maker (= 卖家是 taker,sell-taker),`false` = 卖家是 maker (= 买家是 taker,buy-taker)。

---

## 4. 决策矩阵 (decision matrix)

| flow | 卖墙 | 买方流动性 | 交易频率 | **推荐 setting** |
|---|---|---|---|---|
| > 0 | < $300 | > $200 | > 30/h | **climb 模式** (Walk-Up enabled) |
| > 0 | < $500 | > $100 | > 10/h | **弱 climb** (保守 ETA) |
| -0.3 ~ +0.3 | 任意 | 任意 | 任意 | **自交易 + 弱 climb** |
| < -0.3 | 任意 | 任意 | 任意 | **自交易 only** ([safe-recipe](./safe-recipe.md)) |
| < -0.5 | 任意 | 任意 | 任意 | **自交易 only + tickInterval 240s** (风险减半) |
| 任意 | 任意 | < $100 | < 10/h | **机器人不启动推荐** (市场几近停止) |

---

## 5. 案例: 2026-04-25 IUPUSDT

**测量结果 (canary 机器人启动前):**
- 24h flow score: **-0.81** (买入 $87 / 抛售 $853,强烈卖压)
- 卖墙 (至 $0.005): **$2,792** (我们 USDT $640 无法穿越)
- 买方流动性 (mid 下): **$138** (非常薄)
- 交易频率: **16.6/h** (低)

**判断**: 上表的 `flow < -0.5` → **自交易 only + tickInterval 240s** 应用。

**结果 (11h 运营)**:
- USDT $642 → $658.88 (**+$16.88,+2.63%**)
- 吸收 mid -4.89% 大波动
- 市场 quoteVolume $1,253 → $1,441 (活跃度 ↑)
- 机器人吸收市场波动同时保护 USDT + 维持交易量

→ 市场流向分析 → 保守 setting 选择 → 稳定运营的标准案例。

---

## 6. 分析自动化 (可选)

每小时 4 指标自动测量 + Slack/Telegram 警报:
- flow score < -0.5 变化时警报
- 卖墙变动 ±20% 时警报
- 交易频率 < 10/h 持续时警报

→ 快速应对市场变化,运营决策负担 ↓。

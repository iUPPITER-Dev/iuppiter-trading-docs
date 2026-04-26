# Volume-MM 机器人运营指南

> **对象**: 使用 IUPPITER Volume-MM 机器人产生交易量或调整价格的运营者。
> **前提知识**: 了解交易所 API 密钥 / 订单簿 / 自交易概念即可。

---

## 1. Volume-MM 机器人简介

Volume-MM 机器人结合 **自交易 (self-cross)** 和 **可选的价格爬升 (Walk-Up)** 两种行为,在市场中产生交易量和活跃度:

- **自交易**: 在同一用户的两个钱包 (bid wallet / ask wallet) 之间下达 best-pair 匹配订单 → 双方互相交易 → 计入 MEXC 交易量
- **Walk-Up**: 将 baseline 逐步推向目标价格以移动 mid 的机制 (可启用/禁用)

后台还运行 5-layer **防御系统** (Asymmetric Defense):
- 自动检测异常价格变动、库存偏差、外部抛售压力等 → 自动暂停机器人或将 ladder 调整为保守模式。

---

## 2. 启动前检查清单

### 2.1 资本准备
| 钱包 | 角色 | 推荐余额 |
|---|---|---|
| `T-M1` (bid) | 买方 ladder | USDT $300+ + 部分 IUP |
| `T-M2` (ask) | 卖方 ladder | USDT $300+ + 部分 IUP |
| `T-L` (settle) | 结算 (可选) | 少量 IUP |

### 2.2 市场流向分析 (必须)
**机器人启动前务必** 测量 [market-analysis.md](./market-analysis.md) 的 4 项指标:
1. 24h flow score (买/卖比率)
2. 卖墙 depth (到目标价格的累积 ask USDT)
3. 买方流动性厚度
4. 24h 交易频率

→ flow score < -0.5 (强烈卖压) 时**禁止 climb 尝试**,使用 [safe-recipe.md](./safe-recipe.md) 的自交易 only 模式。

### 2.3 minNotional 检查
MEXC 的 IUPUSDT 最小订单金额为 **$1 USDT**。需保证 order size × 当前 mid ≥ $1。
例: mid $0.004 → 最小 order size 250 IUP,安全起见推荐 350 IUP (= $1.4)。

---

## 3. 机器人创建

在仪表板 (`https://mm.iuppiter.io/bots`) 点击「新建机器人」→ 输入以下项目:

| 项目 | 推荐值 |
|---|---|
| 交易对 | `IUPUSDT` |
| 资本 | USDT $640+ |
| API 密钥 | T-M1 (bid), T-M2 (ask), T-L (settle) |
| Target price | 当前 mid 附近 (climb 困难环境下) |
| Tolerance | 5% |
| Walk-Up | 仅在市场买压强势时 enabled |
| Defense | bias=balanced, enforcement=enforced |
| daily volume | $200~500 USDT |

**详细安全 setting**: [safe-recipe.md](./safe-recipe.md)

---

## 4. 运营监控

### 4.1 仪表板
- `mode`: NORMAL / ENGAGED_BUY / ENGAGED_SELL / DEFENSIVE / CRASH / RECOVERY
- `walkUpState`: INIT / CLIMBING / MAINTAIN / HOLDING / RETREATING / STAGED_CLIMB / PULSE
- `breakers`: 5 种 (NetTaker / PriceMove / StopLoss / InventorySkew / BookSanity)

### 4.2 核心监控指标
| 指标 | 正常 | 警告 | 立即停止 |
|---|---|---|---|
| mode | NORMAL | DEFENSIVE 短暂 | CRASH 持续 |
| breaker trip | 0 | 偶尔 | 1h 内 5 次以上 |
| USDT 变动 | +盈利 / 横盘 | 起始 -$2 | 起始 -$5 |
| fill rate | 0.5~2/min | <0.3/min | 0 (5min 以上) |

### 4.3 推荐检查频率
- **首 1h**: 每 5-10 分钟 close monitoring
- **1h ~ 6h**: 1 小时单位检查 (仪表板)
- **6h+**: 6-12 小时单位或仅在警报触发时

### 4.4 prod log 检查 (必要时)
仪表板 metric 为 0 等异常但余额/订单正常时,参考 [troubleshooting](#) 的 prod log grep 模式。

---

## 5. 停止 + 资本回收

### 5.1 正常停止
- 仪表板「停止机器人」按钮 → backend 对所有 wallet 执行 `cancelAllOrders`
- 或 API: `POST /api/v1/bots/{botId}/stop`

### 5.2 停止后验证
- 所有 wallet 的 open orders = 0
- locked balance = 0 (全部移至 free)
- IUP/USDT 余额相对起始合理变动 (自交易 fee + spread 抽取 ± mid 变动)

### 5.3 资本回收 (必要时)
T-L → 主钱包 `internalTransfer` 或外部提现。但若 IUP 分散在多个钱包,先集中到一个钱包再提现。

---

## 6. 警报响应手册

| 警报 | 立即处理 |
|---|---|
| breaker trip 1 次 | 1h 观察,确认自动恢复 |
| breaker 1h 5 次以上 | 将 tickInterval 增至 240s,再观察 1h |
| USDT -$5 from 起始 | 立即停止,市场分析后重新调整 setting |
| mode CRASH 5min+ | 立即停止,prod log + 外部市场确认 |
| fill_count 1h 内为 0 | prod log 确认 (`bestPairFilled` grep),wiring 问题可能 |
| 外部卖压 wave (mid -10%+) | 推荐停止,稳定后重启 |

---

## 7. 相关资料

- [safe-recipe.md](./safe-recipe.md) — 自交易 only 安全 setting (验证过的配方)
- [market-analysis.md](./market-analysis.md) — 市场流向 4 指标分析
- [incident-2026-04-24.md](./incident-2026-04-24.md) — 事故复盘 + 非对称防御 narrative

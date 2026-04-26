# 市場フロー 4 指標分析 — ボット運用判断ガイド

> **使用タイミング**: 新規ボット起動前 / climb モード有効化前 / ボット setting 変更前。
> **目的**: 市場の流れが私たちの戦略に有利か客観的に判断。

---

## 1. なぜ分析するか

ボット運用失敗事例の大半は「市場が不利なのに無理な setting」で発生:
- 外部買い手不在の環境で climb 試行 → USDT 流出確実
- 厚い売り壁を正面突破 → 売り手に USDT 吸収される
- 取引頻度の低い市場で活性度を強制 → fee のみ累積損失

**4 指標を一緒に見る**と市場の友好度 / リスク度が一目でわかる。

---

## 2. 4 指標定義

### 2.1 24h flow score
**定義**: `(buy_taker_usdt - sell_taker_usdt) / total`  
**解釈**: 外部市場の買い/売りバランス。正 = 買い優勢、負 = 売り優勢。

| 値 | 意味 | ボット戦略 |
|---|---|---|
| `> +0.3` | 買い優勢 (強気) | climb 試行可能 |
| `+0.3 ~ -0.3` | バランス (横ばい) | 自己クロス + 弱い climb |
| `-0.3 ~ -0.5` | 売り優勢 (弱気) | 自己クロス only |
| `< -0.5` | 強い売り (崩壊) | 自己クロスも慎重、USDT 保全 1 順位 |

### 2.2 売り壁 depth
**定義**: 現在の mid から目標価格までの累積 ask USDT。  
**解釈**: 私たちが climb 時に通過する必要がある外部の売り物量。

| 比較 | 意味 |
|---|---|
| 売り壁 USDT < 私たち USDT × 50% | 通過可能 (climb コスト低) |
| 売り壁 USDT ≈ 私たち USDT | リスク (資本ほぼ枯渇) |
| 売り壁 USDT > 私たち USDT × 200% | climb 困難 (正面突破不可) |

### 2.3 買い厚みの厚さ
**定義**: 現在の mid 下の一定区間 (-5%~ -10%) の累積 bid USDT。  
**解釈**: 外部売りが mid を push down する際に支える買い物量。

| 比較 | 意味 |
|---|---|
| 買い厚み > $500 | 安全網十分 |
| 買い厚み $100~500 | 普通 (防御必要時はボット ladder が支え役) |
| 買い厚み < $100 | 非常に薄い、売り時に価格自由落下リスク |

### 2.4 24h 取引頻度
**定義**: 24h 中の外部 trade 数 / 時間。  
**解釈**: 市場活性度。低すぎると私たちの ladder が hit される頻度も低い。

| 頻度 | 意味 |
|---|---|
| > 50/h | 活性 (自己クロス + 外部 hit 頻繁) |
| 10~50/h | 普通 |
| < 10/h | 停滞 (私たちのボットが市場取引量の大半) |

---

## 3. 測定方法 (Node スクリプト例)

### 3.1 Orderbook depth (売り壁 + 買い厚み)

```javascript
const r = await fetch('https://api.mexc.com/api/v3/depth?symbol=IUPUSDT&limit=5000')
const ob = await r.json()
const bid = parseFloat(ob.bids[0][0]), ask = parseFloat(ob.asks[0][0])
const mid = (bid + ask) / 2

// 売り壁: $0.005 までの累積
let askUsdt = 0
for (const [p, q] of ob.asks) {
  const price = parseFloat(p), qty = parseFloat(q)
  if (price <= 0.005) askUsdt += price * qty
}

// 買い厚み: mid 下の累積
let bidUsdt = 0
for (const [p, q] of ob.bids) {
  bidUsdt += parseFloat(p) * parseFloat(q)
}
console.log(`mid=${mid} askWall=$${askUsdt.toFixed(0)} bidDepth=$${bidUsdt.toFixed(0)}`)
```

### 3.2 24h flow score + 取引頻度

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

> MEXC `t.m` フィールド: `true` = 買い手が maker (= 売り手が taker、sell-taker)、`false` = 売り手が maker (= 買い手が taker、buy-taker)。

---

## 4. 判断基準 (decision matrix)

| flow | 売り壁 | 買い厚み | 取引頻度 | **推奨 setting** |
|---|---|---|---|---|
| > 0 | < $300 | > $200 | > 30/h | **climb モード** (Walk-Up enabled) |
| > 0 | < $500 | > $100 | > 10/h | **弱い climb** (保守的 ETA) |
| -0.3 ~ +0.3 | 任意 | 任意 | 任意 | **自己クロス + 弱い climb** |
| < -0.3 | 任意 | 任意 | 任意 | **自己クロス only** ([safe-recipe](./safe-recipe.md)) |
| < -0.5 | 任意 | 任意 | 任意 | **自己クロス only + tickInterval 240s** (リスク半減) |
| 任意 | 任意 | < $100 | < 10/h | **ボット未稼働推奨** (市場ほぼ停止) |

---

## 5. 事例: 2026-04-25 IUPUSDT

**測定結果 (canary ボット起動前):**
- 24h flow score: **-0.81** (買い $87 / 売り $853、強い売り優勢)
- 売り壁 ($0.005 まで): **$2,792** (私たち USDT $640 で通過不可)
- 買い厚み (mid 下): **$138** (非常に薄い)
- 取引頻度: **16.6/h** (低い)

**判断**: 上記表の `flow < -0.5` → **自己クロス only + tickInterval 240s** 適用。

**結果 (11h 運用)**:
- USDT $642 → $658.88 (**+$16.88、+2.63%**)
- mid -4.89% 大変動を吸収
- 市場 quoteVolume $1,253 → $1,441 (活性度 ↑)
- ボットが市場変動を吸収しつつ USDT 保全 + 取引量維持

→ 市場フロー分析 → 保守的 setting 選択 → 安定運用の正解事例。

---

## 6. 分析自動化 (任意)

毎時 4 指標自動測定 + Slack/Telegram アラーム:
- flow score < -0.5 変化時アラーム
- 売り壁変動 ±20% 時アラーム
- 取引頻度 < 10/h 継続時アラーム

→ 市場変化に素早く対応、運用判断負担 ↓。

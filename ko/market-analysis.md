# 시장 흐름 4 지표 분석 — 봇 운영 결정 가이드

> **사용 시점**: 새 봇 가동 전 / climb 모드 활성화 전 / 봇 setting 변경 전.
> **목적**: 시장 흐름이 우리 전략에 우호적인지 객관적으로 판단.

---

## 1. 왜 분석하나

봇 운영 실패 사례 대부분은 "시장이 우리한테 불리한데 무리한 setting" 에서 발생:
- 외부 매수자 부재 환경에서 climb 시도 → USDT 출혈 보장
- 두꺼운 매도벽 정면 돌파 → 매도자에게 USDT 흡수
- 거래 빈도 낮은 시장에서 활성도 강제 → fee 만 누적 손실

**4 지표를 함께 보면** 시장의 우호도 / 위험도가 한눈에 보임.

---

## 2. 4 지표 정의

### 2.1 24h flow score
**정의**: `(buy_taker_usdt - sell_taker_usdt) / total`  
**해석**: 외부 시장의 매수/매도 균형. 양수 = 매수 우세, 음수 = 매도 우세.

| 값 | 의미 | 봇 전략 |
|---|---|---|
| `> +0.3` | 매수 우세 (강세장) | climb 시도 가능 |
| `+0.3 ~ -0.3` | 균형 (횡보) | 자전거래 + 약한 climb 가능 |
| `-0.3 ~ -0.5` | 매도 우세 (약세장) | 자전거래 only |
| `< -0.5` | 강한 매도 (붕괴 흐름) | 자전거래도 신중, USDT 보존 1순위 |

### 2.2 매도벽 depth
**정의**: 현재 mid 부터 목표 가격까지 누적 ask USDT.  
**해석**: 우리가 climb 시 통과해야 할 외부 매도 물량.

| 비교 | 의미 |
|---|---|
| 매도벽 USDT < 우리 USDT × 50% | 통과 가능 (climb 비용 적음) |
| 매도벽 USDT ≈ 우리 USDT | 위험 (자본 거의 소진) |
| 매도벽 USDT > 우리 USDT × 200% | climb 무리 (정면 돌파 불가) |

### 2.3 매수 매물대 두께
**정의**: 현재 mid 아래 일정 구간 (-5%~ -10%) 의 누적 bid USDT.  
**해석**: 외부 매도가 mid 를 push down 할 때 받쳐줄 매수 물량.

| 비교 | 의미 |
|---|---|
| 매수 매물대 > $500 | 안전망 충분 |
| 매수 매물대 $100~500 | 보통 (방어 필요 시 봇 ladder 가 받침 역할) |
| 매수 매물대 < $100 | 매우 얇음, 매도 시 가격 자유낙하 위험 |

### 2.4 24h 거래 빈도
**정의**: 24h 동안 외부 trade 수 / 시간.  
**해석**: 시장 활성도. 너무 낮으면 우리 ladder 가 hit 되는 빈도도 낮음.

| 빈도 | 의미 |
|---|---|
| > 50/h | 활성 (자전거래 + 외부 hit 빈번) |
| 10~50/h | 보통 |
| < 10/h | 정체 (우리 봇이 시장 거래량의 대부분) |

---

## 3. 측정 방법 (Node 스크립트 예시)

### 3.1 Orderbook depth (매도벽 + 매수 매물대)

```javascript
const r = await fetch('https://api.mexc.com/api/v3/depth?symbol=IUPUSDT&limit=5000')
const ob = await r.json()
const bid = parseFloat(ob.bids[0][0]), ask = parseFloat(ob.asks[0][0])
const mid = (bid + ask) / 2

// 매도벽: $0.005 까지 누적
let askUsdt = 0
for (const [p, q] of ob.asks) {
  const price = parseFloat(p), qty = parseFloat(q)
  if (price <= 0.005) askUsdt += price * qty
}

// 매수 매물대: mid 아래 누적
let bidUsdt = 0
for (const [p, q] of ob.bids) {
  bidUsdt += parseFloat(p) * parseFloat(q)
}
console.log(`mid=${mid} askWall=$${askUsdt.toFixed(0)} bidDepth=$${bidUsdt.toFixed(0)}`)
```

### 3.2 24h flow score + 거래 빈도

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

> MEXC `t.m` 필드: `true` = 매수자가 maker (= 매도자가 taker, sell-taker), `false` = 매도자가 maker (= 매수자가 taker, buy-taker).

---

## 4. 판단 기준 (decision matrix)

| flow | 매도벽 | 매수 매물대 | 거래 빈도 | **추천 setting** |
|---|---|---|---|---|
| > 0 | < $300 | > $200 | > 30/h | **climb 모드** (Walk-Up enabled) |
| > 0 | < $500 | > $100 | > 10/h | **약한 climb** (보수적 ETA) |
| -0.3 ~ +0.3 | 무관 | 무관 | 무관 | **자전거래 + 약한 climb** |
| < -0.3 | 무관 | 무관 | 무관 | **자전거래 only** ([safe-recipe](./safe-recipe.md)) |
| < -0.5 | 무관 | 무관 | 무관 | **자전거래 only + tickInterval 240s** (risk 절반) |
| 무관 | 무관 | < $100 | < 10/h | **봇 미가동 권장** (시장 거의 죽음) |

---

## 5. 사례: 2026-04-25 IUPUSDT

**측정 결과 (canary 봇 가동 전):**
- 24h flow score: **-0.81** (매수 $87 / 매도 $853, 강한 매도 우세)
- 매도벽 ($0.005 까지): **$2,792** (우리 USDT $640 으로 통과 불가)
- 매수 매물대 (mid 아래): **$138** (매우 얇음)
- 거래 빈도: **16.6/h** (낮음)

**판단**: 위 표의 `flow < -0.5` → **자전거래 only + tickInterval 240s** 적용.

**결과 (11h 운영)**:
- USDT $642 → $658.88 (**+$16.88, +2.63%**)
- mid -4.89% 큰 변동 흡수
- 시장 quoteVolume $1,253 → $1,441 (활성도 ↑)
- 봇이 시장 변동을 흡수하면서 USDT 보존 + 거래량 유지

→ 시장 흐름 분석 → 보수적 setting 선택 → 안정 운영의 정석 사례.

---

## 6. 분석 자동화 (선택)

매시간 4 지표 자동 측정 + Slack/Telegram 알람:
- flow score < -0.5 변경 시 알람
- 매도벽 변동 ±20% 시 알람
- 거래 빈도 < 10/h 지속 시 알람

→ 시장 변화에 빠르게 대응, 운영 의사결정 부담 ↓.

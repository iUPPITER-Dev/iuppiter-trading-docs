# 자전거래 only 안전 운영 레시피

> **검증된 setting**: USDT 보존 + 거래량 유지를 동시에 달성하는 보수적 운영.
> **검증 기간**: 2026-04-25 ~ 04-26 canary 11h (USDT $642 → $658.88, +$16.88 보존).

---

## 1. 언제 이 레시피를 사용하나

다음 중 하나라도 해당되면 **이 레시피로 시작**:
- 24h flow score < -0.5 (외부 매수자 부재)
- 매도벽 두꺼움 (capital 으로 통과 불가)
- 시장 변동성이 크고 방향 불명
- 새 봇 가동, 안정성 검증부터 하고 싶을 때

→ 이 레시피는 가격 상승을 시도하지 **않습니다**. 자전거래로 거래량 + 시장 활성도만 유지.

---

## 2. Setting 상세 (DB `volume_mm_config`)

```json
{
  "walkUp": { "enabled": false },
  "targetPrice": { "price": <현재 mid>, "tolerancePct": 5 },
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

### 핵심 파라미터 설명

| 파라미터 | 값 | 이유 |
|---|---|---|
| `walkUp.enabled` | `false` | climb 시도 안 함 — USDT 출혈 방지 |
| `targetPrice.price` | 현재 mid | ladder 가 mid 양쪽 좁게 형성 |
| `tolerancePct` | 5 | mid ±5% 안에서만 활동 |
| `tickIntervalMs` | 240000 (240s) | cycle 빈도 절반 → fee cost 절반 → risk 절반 |
| `bestPairSize` / `baseOrderSize` | 350 | $0.0036 mid 에서 $1.26 (minNotional $1 통과) |
| `baseSpreadPct` | 0.003 (0.3%) | bid/ask 충분히 분리 |
| `inventorySkewRatio` | 1 | 시작 IUP 편향 잔고로 인한 즉시 trip 방지 |

---

## 3. 메커니즘 (canary 11h 관측 검증)

봇은 시장 흐름에 따라 자동 균형:

- **mid 하락 흐름** (-1~-5%):
  - 외부 매도자가 우리 ladder 의 bid 측 (mid 위) 을 hit
  - 우리: IUP 매수, USDT 일시 지출 (단기 -)
  - 예: 9h 시점 -$3.15 단기 손실

- **mid 회복 흐름** (+1~+2.5%):
  - 외부 매수자가 우리 ladder 의 ask 측 (mid 아래) 를 hit
  - 우리: IUP 매도, USDT 회수 (+)
  - 예: 10h 시점 +$3.32 회복

- **mid 횡보** (±0.5%):
  - self-cross 만 작동, 자전거래 fee 약간 차감
  - USDT 변화 거의 0 (중립)

→ 시장 횡보/약변동 환경에서 **자전거래 fee + spread 추출로 net positive**.

---

## 4. canary 11h 검증 결과

| 시점 | mid 변화 | USDT (시작 $642 대비) |
|---|---|---|
| 1h | 안정 | **+$18.10** (외부 매수 흡수) |
| 3h (peak) | +0.08% | **+$19.52** ⬆ |
| 6h | -4.89% 누적 | +$6.28 |
| 8h | (240s tick 적용) | +$6.25 |
| 9h 🚨 | -1.45% | **-$3.15** (단기 alarm) |
| 10h | +2.55% 회복 | +$3.32 |
| 11h | -5.76% 변동 | +$1.27 |
| **stop** | (mid 회복) | **+$16.88 최종** ✅ |

**총 6시간 중 NORMAL 100%, breaker trip 0, 시장 변동 흡수, 자본 회수 완료.**

---

## 5. 알람 기준 (이 레시피 적용 시)

| 알람 | 기준 | 조치 |
|---|---|---|
| **L1 주의** | USDT -$3 from 시작 | 1h 관측, 시장 흐름 확인 |
| **L2 경고** | USDT -$5 from 시작 | 즉시 stop 권장 (사전 합의 stop loss) |
| **L3 즉시 stop** | mode CRASH 5min+ 또는 mid -10%+ | 무조건 stop |

---

## 6. 변형 옵션 (필요 시)

| 상황 | 조정 |
|---|---|
| 거래량 더 키우고 싶음 | `tickIntervalMs` 240s → 120s (cycle 2배, risk 2배) |
| Risk 더 줄이고 싶음 | `tickIntervalMs` 240s → 360s 또는 `baseSpreadPct` 0.003 → 0.005 (spread 확대) |
| 매도벽이 가까움 | `tolerancePct` 5 → 3 (band 더 좁게) |
| 시장 매수 등장 | walkUp.enabled=true + targetPrice 조금 위로 (climb 시작) |

---

## 7. 다음 단계

- 봇 안정 운영 확인 (6h+) → 다른 코인 / capital tier 확장
- 시장 매수 등장 → climb 모드 검토 ([market-analysis.md](./market-analysis.md))
- 사고 발생 → [incident-2026-04-24.md](./incident-2026-04-24.md) 참조

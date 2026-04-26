# Volume-MM 봇 운영 가이드

> **대상**: IUPPITER Volume-MM 봇으로 거래량을 만들거나 가격을 조정하려는 운영자.
> **사전 지식**: 거래소 API 키 / 호가 / 자전거래 개념 정도면 충분.

---

## 1. Volume-MM 봇이란

Volume-MM 봇은 **자전거래 (self-cross)** 와 **선택적 가격 상승 (Walk-Up)** 두 가지 동작을 조합해 시장에 거래량과 활성도를 만드는 봇입니다.

- **자전거래**: 같은 사용자의 두 지갑 (bid wallet / ask wallet) 사이에 best-pair 매칭 주문을 발주 → 두 지갑이 서로 거래 → MEXC 거래량으로 기록
- **Walk-Up**: 목표 가격까지 baseline 을 점진적으로 끌어올려 mid 를 이동시키는 메커니즘 (활성/비활성 가능)

추가로 5-layer **방어 시스템** (Asymmetric Defense) 이 백그라운드에서 작동:
- 비정상 가격 변동, 인벤토리 편향, 외부 매도 압력 등을 자동 감지하여 봇을 일시 정지하거나 ladder 를 보수적으로 조정.

---

## 2. 시작 전 체크리스트

### 2.1 자본 준비
| 지갑 | 역할 | 권장 잔고 |
|---|---|---|
| `T-M1` (bid) | 매수 측 ladder | USDT $300+ + 일부 IUP |
| `T-M2` (ask) | 매도 측 ladder | USDT $300+ + 일부 IUP |
| `T-L` (settle) | 정산 (옵션) | 소량 IUP |

### 2.2 시장 흐름 분석 (필수)
**봇 시작 전 반드시** [market-analysis.md](./market-analysis.md) 의 4 지표 측정:
1. 24h flow score (매수/매도 비율)
2. 매도벽 depth (목표 가격까지 누적 ask USDT)
3. 매수 매물대 두께
4. 24h 거래 빈도

→ flow score < -0.5 (강한 매도 우세) 시 **climb 시도 금지**, [safe-recipe.md](./safe-recipe.md) 의 자전거래 only 모드 사용.

### 2.3 minNotional 확인
MEXC 의 IUPUSDT 최소 주문 금액은 **$1 USDT**. order size × 현재 mid ≥ $1 보장 필요.
예: mid $0.004 → order size 최소 250 IUP, 안전하게 350 IUP 권장 (= $1.4).

---

## 3. 봇 생성

대시보드 (`https://mm.iuppiter.io/bots`) 에서 "새 봇 생성" → 다음 항목 입력:

| 항목 | 추천 값 |
|---|---|
| 심볼 | `IUPUSDT` |
| 자본 | USDT $640+ |
| API 키 | T-M1 (bid), T-M2 (ask), T-L (settle) |
| Target price | 현재 mid 근처 (climb 무리 환경에서) |
| Tolerance | 5% |
| Walk-Up | 시장 매수 우세 시만 enabled |
| Defense | bias=balanced, enforcement=enforced |
| daily volume | $200~500 USDT |

**상세 안전 setting**: [safe-recipe.md](./safe-recipe.md)

---

## 4. 운영 모니터링

### 4.1 대시보드
- `mode`: NORMAL / ENGAGED_BUY / ENGAGED_SELL / DEFENSIVE / CRASH / RECOVERY
- `walkUpState`: INIT / CLIMBING / MAINTAIN / HOLDING / RETREATING / STAGED_CLIMB / PULSE
- `breakers`: 5 종 (NetTaker / PriceMove / StopLoss / InventorySkew / BookSanity)

### 4.2 핵심 모니터링 지표
| 지표 | 정상 | 경고 | 즉시 stop |
|---|---|---|---|
| mode | NORMAL | DEFENSIVE 일시 | CRASH 지속 |
| breaker trip | 0 | 가끔 | 1h 동안 5회+ |
| USDT 변동 | +수익 / 횡보 | 시작 -2 USDT | 시작 -5 USDT |
| fill rate | 0.5~2/min | <0.3/min | 0 (5min 이상) |

### 4.3 권장 점검 주기
- **첫 1h**: 5-10 분 단위 close monitoring
- **1h ~ 6h**: 1시간 단위 체크 (대시보드)
- **6h+**: 6-12시간 단위 또는 알람 발동 시만

### 4.4 prod log 확인 (필요 시)
대시보드 metric 이 0 등 비정상이지만 잔고/주문이 정상이면 [troubleshooting](#) 의 prod log grep 패턴 참조.

---

## 5. 정지 + 자본 회수

### 5.1 정상 정지
- 대시보드 "봇 정지" 버튼 → backend 가 모든 wallet 의 `cancelAllOrders` 실행
- 또는 API: `POST /api/v1/bots/{botId}/stop`

### 5.2 정지 후 검증
- 모든 wallet 의 open orders = 0
- locked balance = 0 (모두 free 로 이동)
- IUP/USDT 잔고가 시작 대비 합리적 변동 (자전거래 fee + spread 추출 ± mid 변동)

### 5.3 자본 회수 (필요 시)
T-L → 메인 지갑으로 `internalTransfer` 또는 외부 출금. 단 IUP 가 다른 지갑에 분산돼 있으면 먼저 한 지갑으로 모은 후 출금.

---

## 6. 알람 대응 매뉴얼

| 알람 | 즉시 조치 |
|---|---|
| breaker trip 1회 | 1h 관측, 자동 회복 확인 |
| breaker 1h 5회+ | tickInterval 240s 로 늘리고 1h 더 관측 |
| USDT -$5 from 시작 | 즉시 stop, 시장 분석 후 setting 재조정 |
| mode CRASH 5min+ | 즉시 stop, prod log + 외부 시장 확인 |
| fill_count 1h 동안 0 | prod log 확인 (`bestPairFilled` grep), wiring 문제 가능성 |
| 외부 매도 wave (mid -10%+) | 봇 stop 권장, 안정 후 재가동 |

---

## 7. 추가 자료

- [safe-recipe.md](./safe-recipe.md) — 자전거래 only 안전 setting (검증된 레시피)
- [market-analysis.md](./market-analysis.md) — 시장 흐름 4 지표 분석
- [incident-2026-04-24.md](./incident-2026-04-24.md) — 사고 회고 + 비대칭 방어 narrative

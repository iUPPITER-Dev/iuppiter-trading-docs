# Volume-MM ボット運用ガイド

> **対象**: IUPPITER Volume-MM ボットで取引量を作り出したり、価格を調整したい運用者。
> **前提知識**: 取引所 API キー / 板情報 / 自己クロス取引の概念があれば十分。

---

## 1. Volume-MM ボットとは

Volume-MM ボットは **自己クロス取引 (self-cross)** と **任意の価格上昇 (Walk-Up)** の 2 種類の動作を組み合わせ、市場に取引量と活性度を生み出すボットです。

- **自己クロス**: 同じユーザーの 2 つのウォレット (bid wallet / ask wallet) 間で best-pair マッチング注文を発注 → 双方が互いに約定 → MEXC の取引量として記録
- **Walk-Up**: 目標価格まで baseline を段階的に押し上げて mid を移動させる仕組み (有効/無効の切替可能)

加えて 5-layer の **防御システム** (Asymmetric Defense) がバックグラウンドで稼働:
- 異常な価格変動、在庫の偏り、外部の売り圧力などを自動検知してボットを一時停止または ladder を保守的に調整。

---

## 2. 開始前チェックリスト

### 2.1 資本準備
| ウォレット | 役割 | 推奨残高 |
|---|---|---|
| `T-M1` (bid) | 買い側 ladder | USDT $300+ + 一部 IUP |
| `T-M2` (ask) | 売り側 ladder | USDT $300+ + 一部 IUP |
| `T-L` (settle) | 決済 (任意) | 少量 IUP |

### 2.2 市場フロー分析 (必須)
**ボット開始前に必ず** [market-analysis.md](./market-analysis.md) の 4 指標を測定:
1. 24h flow score (買い/売り比率)
2. 売り壁の depth (目標価格までの累積 ask USDT)
3. 買い厚みの厚さ
4. 24h 取引頻度

→ flow score < -0.5 (強い売り優勢) の場合、**climb 試行禁止**。[safe-recipe.md](./safe-recipe.md) の自己クロス only モードを使用。

### 2.3 minNotional 確認
MEXC の IUPUSDT 最小注文金額は **$1 USDT**。order size × 現在 mid ≥ $1 を保証する必要あり。
例: mid $0.004 → 最小 order size 250 IUP、安全のため 350 IUP 推奨 (= $1.4)。

---

## 3. ボット作成

ダッシュボード (`https://mm.iuppiter.io/bots`) で「新規ボット」→ 以下の項目を入力:

| 項目 | 推奨値 |
|---|---|
| シンボル | `IUPUSDT` |
| 資本 | USDT $640+ |
| API キー | T-M1 (bid), T-M2 (ask), T-L (settle) |
| Target price | 現在 mid 付近 (climb 困難な環境では) |
| Tolerance | 5% |
| Walk-Up | 市場の買い優勢時のみ enabled |
| Defense | bias=balanced, enforcement=enforced |
| daily volume | $200~500 USDT |

**詳細な安全 setting**: [safe-recipe.md](./safe-recipe.md)

---

## 4. 運用モニタリング

### 4.1 ダッシュボード
- `mode`: NORMAL / ENGAGED_BUY / ENGAGED_SELL / DEFENSIVE / CRASH / RECOVERY
- `walkUpState`: INIT / CLIMBING / MAINTAIN / HOLDING / RETREATING / STAGED_CLIMB / PULSE
- `breakers`: 5 種 (NetTaker / PriceMove / StopLoss / InventorySkew / BookSanity)

### 4.2 主要モニタリング指標
| 指標 | 正常 | 警告 | 即停止 |
|---|---|---|---|
| mode | NORMAL | DEFENSIVE 一時的 | CRASH 継続 |
| breaker trip | 0 | たまに | 1h で 5 回以上 |
| USDT 変動 | +利益 / 横ばい | 開始 -$2 | 開始 -$5 |
| fill rate | 0.5~2/min | <0.3/min | 0 (5min 以上) |

### 4.3 推奨確認頻度
- **最初 1h**: 5-10 分間隔で close monitoring
- **1h ~ 6h**: 1 時間単位でチェック (ダッシュボード)
- **6h+**: 6-12 時間単位またはアラーム発動時のみ

### 4.4 prod log 確認 (必要時)
ダッシュボードの metric が 0 など異常だが残高/注文が正常な場合、[troubleshooting](#) の prod log grep パターン参照。

---

## 5. 停止 + 資本回収

### 5.1 通常停止
- ダッシュボード「ボット停止」ボタン → backend がすべての wallet の `cancelAllOrders` 実行
- または API: `POST /api/v1/bots/{botId}/stop`

### 5.2 停止後の検証
- すべての wallet の open orders = 0
- locked balance = 0 (すべて free に移動)
- IUP/USDT 残高が開始時から合理的な変動 (自己クロス fee + spread 抽出 ± mid 変動)

### 5.3 資本回収 (必要時)
T-L → メイン wallet へ `internalTransfer` または外部出金。ただし IUP が複数 wallet に分散している場合、まず 1 つに集約後出金。

---

## 6. アラーム対応マニュアル

| アラーム | 即時対応 |
|---|---|
| breaker trip 1 回 | 1h 観測、自動回復確認 |
| breaker 1h 5 回以上 | tickInterval 240s に増やし 1h 追加観測 |
| USDT -$5 from 開始 | 即停止、市場分析後 setting 再調整 |
| mode CRASH 5min+ | 即停止、prod log + 外部市場確認 |
| fill_count 1h で 0 | prod log 確認 (`bestPairFilled` grep)、wiring の問題可能性 |
| 外部の売り wave (mid -10%+) | ボット停止推奨、安定後再起動 |

---

## 7. 関連資料

- [safe-recipe.md](./safe-recipe.md) — 自己クロス only 安全 setting (検証済みレシピ)
- [market-analysis.md](./market-analysis.md) — 市場フロー 4 指標分析

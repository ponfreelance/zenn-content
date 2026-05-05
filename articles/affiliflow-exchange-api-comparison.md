---
title: "自動売買Botを作って分かった日本の取引所API比較 ── 実装者視点の本音レビュー"
emoji: "🔧"
type: "tech"
topics:
  - "Python"
  - "FastAPI"
  - "個人開発"
published: true
---
## 評価軸

取引所APIの比較記事は山ほどあるが、大半は公式サイトの表をコピーしただけ。

この記事は **実際にBot（FastAPI + ccxt）を組んで全取引所に接続した結果** 分かった「実装者視点の違い」をまとめる。評価軸はこの4つ。

| 評価軸 | 意味 |
|--------|------|
| 注文の通しやすさ | 成行注文が使えるか、独自仕様のハンドリングが必要か |
| データ取得の充実度 | OHLCV（ローソク足）・約定履歴が取れるか |
| テスト環境 | サンドボックス／テストネットがあるか |
| 手数料 | Maker/Takerの実コスト |

「APIドキュメントが読みやすいか」みたいな主観は入れない。コードで測れる違いだけを書く。

## 比較表

### 注文まわり

| 取引所 | 成行注文 | 約定確認方法 | 特記事項 |
|--------|---------|-------------|---------|
| **Coincheck** | 指値フォールバック | fetchMyTrades | 手数料0%が魅力 |
| **GMOコイン** | 指値フォールバック | fetchMyTrades | ccxt対応が不完全 |
| **bitFlyer** | 指値フォールバック | fetchMyTrades | 取引量割引あり |
| **Zaif** | 板追随（独自実装必要） | fetchClosedOrders（時刻照合） | order_id=0問題あり |
| **bitbank** | 対応 | fetchMyTrades | Makerリベートあり |
| **Binance Japan** | 対応 | fetchMyTrades | 海外版と同等の安定性 |

Zaifだけ異質。成行注文APIがないため、指値注文を出して板の動きを30秒追跡し、未約定分をキャンセルして再発注する「板追随ロジック」を自前で実装する必要がある。

```python
# Zaifの板追随：指値→30秒待機→キャンセル→ベストプライスで再発注
order = await create_limit_order(symbol, side, amount, best_price)
await asyncio.sleep(30)

if order["remaining"] > 0:
    await cancel_order(order["id"], symbol)
    # ベストask/bidで再発注
    new_price = ticker["ask"] if side == "buy" else ticker["bid"]
    await create_limit_order(symbol, side, order["remaining"], new_price)
    await asyncio.sleep(10)
```

一方Binance JapanやbitbankはccxtのcreateMarketOrderがそのまま使える。実装コストが段違い。

### データ取得

| 取引所 | OHLCV（ローソク足） | 代替手段 | 使える戦略 |
|--------|-------------------|---------|-----------|
| **Coincheck** | 非対応 | CoinGecko経由 | DCA・スキャルピングのみ |
| **GMOコイン** | 非対応 | CoinGecko経由 | DCA・裁定のみ |
| **bitFlyer** | 非対応 | CoinGecko経由 | DCA・スキャルピングのみ |
| **Zaif** | 一部対応（BTC/JPYのみ） | CoinGecko経由 | DCA・スキャルピングのみ |
| **bitbank** | 全ペア対応 | 不要 | ルールベース・グリッド含む全戦略 |
| **Binance Japan** | 全ペア対応 | 不要 | 全戦略 |

OHLCVが取れないとMA（移動平均）クロスやMACD判定ができない。つまりルールベース戦略やグリッド戦略が使えない。

```python
# OHLCV未対応の取引所はDCAにフォールバック
if exchange_name in ["Zaif", "bitFlyer", "Coincheck"]:
    signal = get_dca_signal(bot, symbol, ticker)  # ローソク足なしで動く戦略
else:
    ohlcv = await fetch_ohlcv(symbol, timeframe="1h")
    signal = get_rule_signal(bot, symbol, ohlcv)  # MA/RSI/MACD判定
```

### テスト環境

| 取引所 | サンドボックス | 影響 |
|--------|-------------|------|
| **Coincheck** | なし | 本番でテストするしかない |
| **GMOコイン** | なし | 本番でテストするしかない |
| **bitFlyer** | なし | 本番でテストするしかない |
| **Zaif** | なし | 本番でテストするしかない |
| **bitbank** | あり | 安全にテスト可能 |
| **Binance Japan** | あり | 安全にテスト可能 |

国内取引所は**全滅**。テストネットがないので、最小ロットで本番環境に注文を出すしかない。Botのバグで意図しない注文が入るリスクがある。

自分は日次損失リミット・ドローダウンリミット・連続損失ストップなどの安全装置を多重に入れることで対処している。

### 手数料

| 取引所 | Taker（成行） | Maker（指値） | 特徴 |
|--------|-------------|-------------|------|
| **Coincheck** | 0.0% | 0.0% | 完全無料 |
| **Zaif** | 0.1% | 0.0% | Maker無料 |
| **GMOコイン** | 0.05〜0.09% | -0.01〜-0.03% | Makerはリベート（もらえる） |
| **bitbank** | 0.12% | -0.02% | Makerはリベート |
| **bitFlyer** | 0.15% | 0.15% | 取引量で変動 |
| **Binance Japan** | 0.1% | 0.1% | シンプル |

Coincheckの完全無料は強い。高頻度取引なら手数料差は効いてくるので、スキャルピング系はCoincheckかZaif（Maker）が有利。

bitbankとGMOコインの「負のMaker手数料（リベート）」は指値注文を置くだけでお金がもらえる。マーケットメイキング戦略との相性がいい。

## 結論

「どの取引所が一番いいか」は戦略による。自分が実装して得た結論はこう。

| やりたいこと | おすすめ |
|-------------|---------|
| **とにかく簡単に始めたい** | Binance Japan（API安定・テストネットあり・全戦略対応） |
| **国内で手数料を抑えたい** | Coincheck（完全無料）またはZaif（Maker無料） |
| **MA/MACD使ったルールベース戦略** | bitbank or Binance Japan（OHLCV対応必須） |
| **マーケットメイキング** | bitbank or GMOコイン（Makerリベートあり） |
| **実装の手間を最小にしたい** | Binance Japan（ccxtの標準機能がそのまま動く） |

一つ言えるのは、**ccxtで抽象化されていても取引所ごとの差異は消えない**ということ。特にZaifは独自実装が大量に必要になる。

初めてBot開発に挑戦するなら、API安定性とテストネット対応の観点でBinance Japanが一番ストレスが少ない。国内取引所に絞るならCoincheckが手数料無料＋APIドキュメントが日本語で読みやすく、始めやすい。

### 確定申告の準備もお忘れなく

取引所を複数使い分けてBot運用すると、取引件数が爆増する。確定申告の損益計算は手動では不可能なので、ツールに任せるのが正解。

- [面倒な仮想通貨の損益計算を全自動で【CRYPTACT（クリプタクト）】](https://px.a8.net/svt/ejp?a8mat=4B1EPO+16V8C2+4DGW+5ZMCI)

---

取引所のAPI比較を実際にやってみたい人は、まずAPIキーを発行して実際のレスポンスを見るのが最速の学習法。確定申告の損益計算も忘れずに → [利用者No1の仮想通貨税金計算サービス【CRYPTACT（クリプタクト）】](https://px.a8.net/svt/ejp?a8mat=4B1EPO+16V8C2+4DGW+5YJRM)

---
title: "Claude Codeで仮想通貨自動売買Botを作った話 ── 4週間・601コミット・15取引所対応の全記録"
emoji: "🔧"
type: "tech"
topics:
  - "Python"
  - "FastAPI"
  - "個人開発"
published: true
---
## システム概要

Claude Codeを使って、仮想通貨の自動売買Botを4週間で作った。

スマホアプリからBotの起動・停止ができて、15の取引所に対応、7種類の売買戦略を切り替えられる。デプロイ費用は月0円（Render無料プラン）。

```
スマホアプリ（React Native）
    ↕ REST API / WebSocket
FastAPIバックエンド（Python）
    ↕ ccxt（統一API）
15の仮想通貨取引所
```

最初は「APIで注文を出すスクリプト」くらいのつもりだった。気づいたら600コミット超えてた。

### 主なスペック

| 項目 | 数値 |
|------|------|
| 対応取引所 | 15（国内6 + 海外9） |
| 売買戦略 | 7種類 |
| テストケース | 229個 |
| バックエンド | Pythonファイル81個 |
| フロントエンド | TypeScript/TSXファイル25個 |
| 総コミット数 | 601（4週間） |
| デプロイ費用 | 月0円 |

## 技術スタック

### バックエンド

```
FastAPI 0.111.0    ── APIサーバー
SQLAlchemy 2.0     ── ORM（PostgreSQL）
ccxt 4.x           ── 取引所統一API
APScheduler 3.10   ── Bot定期実行
Groq SDK           ── AI戦略（LLM判定）
Stripe             ── 決済
```

なぜFastAPIか。**async/awaitが標準**だから。取引所APIは応答に1〜3秒かかる。同期処理だとBot5台動かすだけで15秒待ちが発生する。FastAPIなら並列に処理できる。

### フロントエンド

```
React Native 0.81.5  ── モバイルUI
Expo SDK 54           ── ビルド・デプロイ
Zustand               ── 状態管理
React Query 5.40      ── API通信
```

React Nativeを選んだのは「1コードベースでiOS/Android両対応」だから。一人で作ってるのでネイティブ2本メンテは無理。

### デプロイ

```yaml
# render.yaml（これだけで動く）
services:
  - type: web
    name: autotrader-api
    runtime: python
    plan: free
    buildCommand: pip install -r requirements.txt
    startCommand: gunicorn main:app -w 1 -k uvicorn.workers.UvicornWorker
```

Render無料プランは15分でスリープするが、UptimeRobotで5分ごとにpingを飛ばして回避している。

## 実装ポイント

### 7つの売買戦略

全部で1,637行。戦略ごとにモジュールを分けている。

| 戦略 | 概要 | 判定ロジック |
|------|------|-------------|
| ルールベース | テクニカル指標の多数決 | MA + RSI + MACD（3つ中2つ一致で売買） |
| グリッド | レンジ相場向け | 価格帯を分割して自動売買 |
| DCA | 積立型 | 一定間隔で買い続ける |
| AI/LLM | Groqでトレンド分析 | LLMに市場データを渡して判定 |
| 裁定取引 | 取引所間の価格差 | スプレッドが閾値超えたら実行 |
| スキャルピング | 薄利多売 | 板の厚みを見て即時売買 |
| マーケットメイキング | スプレッド獲得 | 買い・売り両方に指値を置く |

ルールベース戦略は「3つの指標の多数決」方式。1つだけのシグナルでは動かない。

```python
# 3つのテクニカル指標で多数決
signals = []
signals.append(get_ma_signal(ohlcv))     # 移動平均クロス
signals.append(get_rsi_signal(ohlcv))     # RSI（過熱判定）
signals.append(get_macd_signal(ohlcv))    # MACD

# 3つ中2つ以上が同じ方向 → 売買実行
buy_count = signals.count("BUY")
sell_count = signals.count("SELL")

if buy_count >= 2:
    return Signal(action="BUY")
elif sell_count >= 2:
    return Signal(action="SELL")
else:
    return Signal(action="HOLD")  # 意見が割れたら何もしない
```

### 安全装置（8層の防御）

自動売買で一番怖いのはバグで資金が溶けること。8層のセーフティを入れた。

```
1. 日次損失リミット（デフォルト500 USDT超で自動停止）
2. 週次損失リミット（デフォルト2,000 USDT超で自動停止）
3. ドローダウン制限（戦略ごとに5〜30%）
4. 連続損失ストップ（5連敗で自動停止）
5. 1日の注文数上限（50注文/日）
6. シグナル確認（同じ方向のシグナルが2回連続で初めて発注）
7. 異常検知（価格5%以上の急変動で取引停止）
8. ポジション照合（30秒ごとに取引所の実ポジションとDB記録を突合）
```

特に7番の異常検知は重要。APIが変な値を返すこともある。直近ティッカーと5%以上乖離していたら「ANOMALY_DETECTED」としてBotを一時停止する。

```python
# 価格の急変動を検知
if abs(current_price - last_price) / last_price > 0.05:
    bot.status = "ANOMALY_DETECTED"
    logger.warning(f"異常検知: {last_price} → {current_price} (乖離率{deviation}%)")
    notify_slack(f"異常検知: {bot.name}")
    return  # 取引しない
```

### 取引所ごとの差異を吸収する仕組み

ccxtで抽象化されていても、取引所ごとに挙動が違う。主な差異と対処法：

| 差異 | 対処 |
|------|------|
| Zaifは成行注文非対応 | 板追随ロジック（指値→30秒待機→再発注） |
| 国内3社はOHLCV非対応 | CoinGecko APIにフォールバック |
| 国内4社はテストネットなし | 最小ロット+安全装置で本番テスト |
| 約定確認方法が異なる | 3段階フォールバック（注文一覧→取引履歴→残高差分） |

これが一番大変だった。取引所を1つ追加するたびに「この取引所はfetchMyTrades使える？」「成行注文の返り値の型は？」をいちいち確認する必要がある。

## 結果

4週間で以下ができた：

- 15取引所対応の自動売買Bot
- スマホから操作できるUI
- 7つの売買戦略を切り替え可能
- 229個のテストケース
- 月0円でデプロイ

Claude Codeが特に役立ったのは以下の場面：

- **FastAPIのルーター設計** — 20以上のAPIエンドポイントを一気に生成
- **ccxtの取引所固有ハンドリング** — 各取引所のドキュメントを読んでエッジケースを提示
- **テスト量産** — 既存コードからテストケースを自動生成
- **デバッグ** — スタックトレースを貼るだけで原因と修正案が出てくる

一人で全部書いたわけじゃない。Claude Codeとペアプロした結果、この速度で出せた。

個人開発でAIツールを使わないのは、もはやハンデだと思う。

---

自動売買Botで取引回数が増えると、確定申告の損益計算が手動では不可能になる。取引履歴をアップロードするだけで自動計算してくれるクリプタクトを使っておくのがおすすめ。

→ [利用者No1の仮想通貨税金計算サービス【CRYPTACT（クリプタクト）】](https://px.a8.net/svt/ejp?a8mat=4B1EPO+16V8C2+4DGW+5YJRM)

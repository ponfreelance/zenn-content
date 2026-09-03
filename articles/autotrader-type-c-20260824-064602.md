---
title: "まず『発注しないボット』を作る — ペーパートレードから始める自動売買"
emoji: "🔧"
type: "tech"
topics:
  - "Python"
  - "FastAPI"
  - "個人開発"
published: true
---
> 本記事は一部アフィリエイトリンク（PR）を含みます

## この記事でできるようになること

C言語は書けるけど自動売買は完全に未経験、という人向けに「実際の注文は一切出さない、値動きを見てログを吐くだけのボット」を作るところまでを解説する。いきなり本番の取引所APIに繋いで発注させると、ロジックのバグで想定外の注文が飛ぶ事故が起きる。だから最初の1本は「発注しない」。これが遠回りに見えて一番早い。この記事を読み終える頃には、価格を定期取得してログに残すだけの最小構成が手元で動いている状態になる。

## 前提知識

### 必要なもの
- 何らかのプログラミング言語で変数・関数・ループが書ける（C言語経験があれば十分）
- ターミナル（コマンドライン）操作の基礎：`cd`、ファイル実行くらい

### 必要ないもの
- 暗号資産の売買経験：この記事では一切売買しないので心配無用
- 金融工学やテクニカル分析の知識：ここでは扱わない。移動平均線が何かも後回しでいい
- Web開発の経験：HTTPが何かをふわっと知っていれば十分

### あると理解が早いもの
- Pythonの文法：C言語が書ければ1〜2時間で読み書きできるようになる。今回はPythonで書く
- 取引所のWebサイトを一度見たことがある：チャート画面のイメージがあると理解が早い

## ステップ1：準備

### やること
Pythonの実行環境を用意し、取引所の公開API（無料・登録不要で使えるもの）で現在価格を取得できることを確認する。

### 手順

1. Pythonをインストールする（すでに入っていれば飛ばす）
   ターミナルで `python3 --version` を打って `Python 3.x.x` と表示されればOK
   （※ 入っていない場合はOS別のインストール手順のスクリーンショットを挿入）

2. HTTPリクエスト用ライブラリをインストールする
   ```bash
   pip3 install requests
   ```

3. 取引所の公開APIを叩いてみる（例：GMOコインの公開API、APIキー不要）
   ```python
   import requests

   res = requests.get("https://api.coin.z.com/public/v1/ticker?symbol=BTC")
   print(res.json())
   ```

4. 実行する
   ```bash
   python3 check_price.py
   ```

### 成功の目安
ターミナルにBTCの現在価格を含んだJSON（辞書のような文字列）が表示されればここまでは正解。

### もしうまくいかない場合
- `ModuleNotFoundError: No module named 'requests'` → ステップ2のインストールを忘れている
- 何も表示されず固まる → ネット接続を確認。社内・学校のネットワークでAPI通信がブロックされていることがある
- `SSLError` が出る → Pythonのバージョンが古い可能性が高い。3.9以上を推奨

## ステップ2：最初の実行（価格を定期取得してログに残す）

### やること
1分おきに価格を取得し、ファイルに書き込み続けるだけのスクリプトを作る。これが「発注しないボット」の正体。

### 手順

1. 以下のコードを `logger_bot.py` として保存する
   ```python
   import requests
   import time
   from datetime import datetime

   def get_price():
       res = requests.get("https://api.coin.z.com/public/v1/ticker?symbol=BTC")
       data = res.json()
       return data["data"][0]["last"]

   while True:
       now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
       price = get_price()
       line = f"{now}, {price}\n"
       print(line, end="")
       with open("price_log.csv", "a") as f:
           f.write(line)
       time.sleep(60)
   ```

2. 実行する
   ```bash
   python3 logger_bot.py
   ```

3. `Ctrl+C` でいつでも止められることを確認する

### 成功の目安
`price_log.csv` というファイルが同じフォルダにでき、1分ごとに1行ずつ「日時, 価格」が追記されていく。数分放置して行数が増えていればOK。

### もしうまくいかない場合
- ファイルができない → 実行しているフォルダの書き込み権限を確認する。管理者権限が必要な場所で実行していないか
- 価格が全く同じ値で止まる → APIのレスポンスをキャッシュしていないか、`get_price()` を毎回呼んでいるか確認
- そもそも起動直後に落ちる → `data["data"][0]["last"]` の構造がAPI側の仕様変更で変わっている可能性。`print(data)` を挟んで実際のJSON構造を見る

## ステップ3：応用（「発注しそうになったら止める」ロジックを足す）

### やること
実際には発注しないが、「もし発注するならこのタイミングだった」というログだけを追加で吐く。これで自分の売買ロジックの妥当性を、お金をかけずに検証できる。

### 手順

1. 単純な条件（例：直前より1%以上下がったら「買いシグナル」とみなす）を追加する
   ```python
   prev_price = None

   while True:
       now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
       price = get_price()

       signal = ""
       if prev_price is not None:
           change = (price - prev_price) / prev_price * 100
           if change <= -1.0:
               signal = "BUY_SIGNAL（実際には発注しない）"

       line = f"{now}, {price}, {signal}\n"
       print(line, end="")
       with open("price_log.csv", "a") as f:
           f.write(line)

       prev_price = price
       time.sleep(60)
   ```

2. 数時間〜1日放置し、シグナルがどのくらいの頻度で出るか観察する

### 成功の目安
`price_log.csv` に `BUY_SIGNAL` の行が時々混ざるようになる。これが出た瞬間に「本当に買っていたら今いくら儲かった／損したか」を後から手計算できる状態になっていればゴール。

### もしうまくいかない場合
- シグナルが一切出ない → 閾値（`-1.0`）が厳しすぎる。まずは `-0.1` くらいに緩めて動作確認する
- シグナルが出すぎる → 逆に閾値を厳しくする。値自体に正解はなく、後で検証しながら調整するもの

## よくあるエラーとその対処

1. **`ConnectionError`**：ネットワークが一時的に切れた、またはAPI側が落ちている。`try/except` で囲んでリトライする処理を足すと安定する
2. **`KeyError: 'data'`**：APIのレスポンス構造が想定と違う。取引所側の仕様変更、または通貨ペアの指定ミス（`symbol=BTC` の綴りなど）
3. **ログファイルが肥大化して開けなくなる**：1分間隔で1週間動かすと1万行を超える。定期的にファイルを分割するか、そもそもDBに移行することを検討する時期のサイン
4. **PCをスリープさせて止まっていた**：ノートPCで動かす場合、スリープでスクリプトごと止まる。長時間動かすならクラウドの小さいサーバー（VPS）に置く選択肢もある

## 次に学ぶべきこと

この記事で「発注しないボット」が動かせたら、次のステップ：

1. **バックテスト（過去データでの検証）**：ためたCSVを使って「このロジックなら過去1ヶ月でいくら儲かっていたか」を計算する
   推奨リソース：Pythonの `pandas` ライブラリの入門記事・書籍

2. **APIキーを使った認証付きAPIの扱い**：価格取得だけでなく、口座残高や注文履歴を取得できるようになる（まだ発注はしない）
   推奨リソース：利用する取引所の公式APIドキュメント

3. **実際の発注（ごく少額から）**：ここで初めて本物の注文を出す。最初は生活に影響のない金額、失っても構わない額で始める
   推奨リソース：取引所の「デモ取引」機能があれば先にそちらで練習

1ヶ月かけて3ステップ進めば、自動売買の基礎は固まる。

## 慣れてきた人へ

ここまでのロジックが固まってきて、C言語での実装経験を活かしてもう少し本格的に組みたくなったら、こういうツールもある：

- **[AutoTrader 実装学習キット](https://autotrader.ponfreelance.com/sales/?utm_source=zenn&utm_medium=article&utm_campaign=autotrader&utm_content=affiliflow)**：FastAPI × React Native で外部APIと連携するアプリの実装を学べる教材
  （まだペーパートレードすら始めていないなら、先にこの記事のステップ2・3を自分の手で動かしてから検討を）

## 関連リソース

自動売買を本格的に始めるなら、まず取引所の口座と、公式APIドキュメントを一通り読める状態を作っておくと後が早い。

- **GMOコイン** [PR]
  この記事で使った公開APIと同じ取引所。口座開設しておくと、次の「認証付きAPI」の段階にそのまま進める。
  → [詳細](https://af.moneypartners.co.jp/...)

- **bitFlyer** [PR]
  APIドキュメントが読みやすく、初めて取引所APIを触る人向けの解説記事も多い。
  → [詳細](https://af.moneypartners.co.jp/...)

## 試すなら

AutoTrader 実装学習キット を実際に触ってみるなら。

→ [AutoTrader 実装学習キット](https://autotrader.ponfreelance.com/sales/?utm_source=zenn&utm_medium=article&utm_campaign=autotrader&utm_content=affiliflow)

---

**著者**：ぽん（@pon_freelance）
C言語実務23年、組み込み／制御系。
副業で技術記事販売と自作ツール販売をやっている。

書いているもの：
- AutoTrader 実装学習キット - FastAPI × React Native で作る外部 API 連携アプリ実装学習キット
（その他：（なし））

**開発の裏側を購読できます** — AutoTrader のリリースごとに「何を・なぜ・どう変えたか」を 2,000〜4,000 字で書き残しています。バグの原因、取引所 API 変更への追従、設計判断のトレードオフまで。
→ [AutoTrader開発ログ（月500円・いつでも解約可）](https://note.com/clab_jp/membership)

---
title: "C言語エンジニアが暗号資産自動売買ボットを作る：最初の一歩"
emoji: "🔧"
type: "tech"
topics:
  - "Python"
  - "FastAPI"
  - "個人開発"
published: false
---
> 本記事は一部アフィリエイトリンク（PR）を含みます

---

C言語でポインタを操れるなら、暗号資産ボットの基礎は思ったより早く掴める。この記事では「取引所のAPIを叩いて価格を取得し、条件に応じて注文を出す」という最小構成のボットをPythonで動かすところまでを扱う。「なぜPythonか」は後述するが、C言語の知識があれば読み替えながら理解できる。

最初の3日は概念の違いに戸惑うかもしれない。でも1週間もあれば、最初のボットが自律的に動く感覚は掴める。

---

## 前提知識

### 必要なもの

- **プログラミング基礎**（変数・ループ・条件分岐の概念）：C言語経験者なら問題なし
- **ターミナル操作**（コマンドを打てる程度）：`gcc`を叩いたことがあれば十分
- **Python 3.10以上**（インストール済み）：後述の手順でインストール

### 必要ないもの

- **金融・経済の専門知識**：この記事では使わない。「高く売って安く買う」が分かれば動かせる
- **Webフレームワークの知識**：FastAPIもDjangoも今回は不要
- **機械学習の知識**：最初のボットはif文で動く

### あると理解が早いもの

- **HTTP通信の基礎**（GETリクエストとは何か）：C言語でソケット通信をやったことがあれば、REST APIの概念は30分で掴める
- **JSON形式の読み方**：取引所のレスポンスはほぼJSONなので、構造を知っていると楽

---

## なぜPythonか（C言語エンジニアへの補足）

率直に言う。取引所の公式SDKとコミュニティのライブラリはPythonに集中している。C言語でREST APIを叩くことは技術的に可能だが、`libcurl`のセットアップ、JSONパーサの選定、エラーハンドリングの実装に時間を取られ、「ボットを作る」という本題から外れる。

最初の1ヶ月はPythonで動かし、仕組みを理解してからC言語に移植するルートが現実的だ。C言語の思考（メモリ管理、処理の流れの明示性）はPythonコードを読むときに確実に活きる。

---

## ステップ1：環境を整える

### やること

取引所アカウントの作成と、Python実行環境の構築を行う。

### 手順

**1. 取引所アカウントを作る**

国内の主要取引所はどこでも構わないが、APIドキュメントが日本語で整備されているCoincheckかbitFlyerが最初には向いている。

<!-- スクリーンショット挿入ポイント：Coincheck新規登録画面 -->

アカウント作成後、本人確認（KYC）を完了させる。審査に数日かかる場合があるので、先にPython環境を整えておくと時間を無駄にしない。

**2. APIキーを発行する**

取引所の管理画面から「APIキー」または「API設定」を探す。

- **APIキー**（公開鍵に相当）
- **シークレットキー**（秘密鍵に相当）

の2つが発行される。C言語の公開鍵暗号と同じ構造だと思えばいい。

> ⚠️ シークレットキーは一度しか表示されない。必ずメモ帳などに控えること。GitHubに絶対にアップロードしないこと。

**3. Pythonをインストールする**

```bash
# macOS（Homebrewがある場合）
brew install python@3.11

# Ubuntu/Debian
sudo apt update && sudo apt install python3.11 python3-pip

# Windowsは python.org からインストーラをダウンロード
```

バージョン確認：

```bash
python3 --version
# Python 3.11.x と表示されればOK
```

**4. 必要なライブラリをインストールする**

```bash
pip3 install requests python-dotenv
```

- `requests`：HTTP通信ライブラリ（C言語でいう`libcurl`の役割）
- `python-dotenv`：APIキーを環境変数で管理するため

### 成功の目安

`python3 --version`でバージョンが表示され、`pip3 install`がエラーなく完了すれば準備完了。

### もしうまくいかない場合

- **`pip3: command not found`**：`python3 -m pip install requests python-dotenv`で代替できる
- **`Permission denied`**：`pip3 install --user requests python-dotenv`を試す
- **Windows でパスが通らない**：インストール時に「Add Python to PATH」にチェックが入っていたか確認

---

## ステップ2：価格を取得してみる

### やること

取引所のAPIを叩いて、ビットコインの現在価格を取得するコードを書く。売買は一切しない。まず「データを読む」ことに集中する。

### 手順

**1. プロジェクトディレクトリを作る**

```bash
mkdir crypto_bot
cd crypto_bot
```

**2. APIキーを環境変数ファイルに書く**

`.env`ファイルを作成（このファイルはGitに含めない）：

```
COINCHECK_API_KEY=ここにAPIキーを貼る
COINCHECK_SECRET_KEY=ここにシークレットキーを貼る
```

**3. 価格取得スクリプトを書く**

`get_price.py`として保存：

```python
import requests

def get_btc_price():
    """
    Coincheck のパブリックAPIからBTC/JPY価格を取得する
    認証不要のエンドポイントを使用
    """
    url = "https://coincheck.com/api/ticker"
    
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()  # HTTPエラーがあれば例外を投げる
        
        data = response.json()
        last_price = data["last"]  # 最終取引価格
        
        return float(last_price)
    
    except requests.exceptions.Timeout:
        print("エラー：タイムアウト。ネットワーク接続を確認してください")
        return None
    except requests.exceptions.RequestException as e:
        print(f"エラー：{e}")
        return None

if __name__ == "__main__":
    price = get_btc_price()
    if price:
        print(f"BTC/JPY 現在価格：{price:,.0f} 円")
```

**4. 実行する**

```bash
python3 get_price.py
# BTC/JPY 現在価格：X,XXX,XXX 円
```

<!-- スクリーンショット挿入ポイント：ターミナルで価格が表示された状態 -->

**C言語エンジニア向け補足**：`response.json()`はJSONを辞書型（連想配列）に変換する。`data["last"]`はCの構造体メンバアクセスに近いが、キーは文字列。`try/except`はCの戻り値チェックに相当する。

### 成功の目安

実際のBTC価格（数百万円台）が表示されれば成功。数字が出ればAPIとの通信は確立している。

### もしうまくいかない場合

- **JSONDecodeError**：APIの仕様変更の可能性。Coincheckのドキュメントでエンドポイントを確認
- **ConnectionError**：ネットワーク接続またはファイアウォールの問題
- **`last`キーがない**：`print(data)`で返ってきたJSONの全体を確認する

---

## ステップ3：条件に応じて動作を変える（簡易ボットの骨格）

### やること

「価格がXX円を下回ったら買い、YY円を上回ったら売り」という最小限のロジックを実装する。**実際の注文は出さず**、まずシミュレーション（ログ出力）で動かす。

### 手順

**1. シンプルなルールベースボットを書く**

`simple_bot.py`として保存：

```python
import time
import requests
from datetime import datetime

# 売買ルール（まずはシミュレーション用に設定）
BUY_THRESHOLD  = 8_000_000   # この価格以下で「買い」シグナル
SELL_THRESHOLD = 10_000_000  # この価格以上で「売り」シグナル
CHECK_INTERVAL = 60          # 何秒ごとに価格を確認するか

def get_btc_price():
    url = "https://coincheck.com/api/ticker"
    try:
        r = requests.get(url, timeout=10)
        r.raise_for_status()
        return float(r.json()["last"])
    except Exception as e:
        print(f"価格取得失敗: {e}")
        return None

def evaluate_signal(price):
    """
    価格を受け取り、売買シグナルを返す
    戻り値: "BUY" / "SELL" / "HOLD"
    """
    if price <= BUY_THRESHOLD:
        return "BUY"
    elif price >= SELL_THRESHOLD:
        return "SELL"
    else:
        return "HOLD"

def run_bot():
    print("ボット起動。Ctrl+Cで停止します。")
    print(f"買いシグナル：{BUY_THRESHOLD:,}円以下")
    print(f"売りシグナル：{SELL_THRESHOLD:,}円以上")
    print("-" * 40)
    
    while True:
        now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        price = get_btc_price()
        
        if price is None:
            print(f"[{now}] 価格取得スキップ")
        else:
            signal = evaluate_signal(price)
            print(f"[{now}] 価格: {price:>12,.0f}円  シグナル: {signal}")
            
            # ここに実際の注文処理を後から追加する
            # if signal == "BUY":
            #     place_order("buy", amount=0.001)
            # elif signal == "SELL":
            #     place_order("sell", amount=0.001)
        
        time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    run_bot()
```

**2. 実行する**

```bash
python3 simple_bot.py
```

出力例：

```
ボット起動。Ctrl+Cで停止します。
買いシグナル：8,000,000円以下
売りシグナル：10,000,000円以上
----------------------------------------
[2024-01-15 10:00:00] 価格:    9,250,000円  シグナル: HOLD
[2024-01-15 10:01:00] 価格:    9,248,500円  シグナル: HOLD
```

<!-- スクリーンショット挿入ポイント：ボットが60秒ごとにログを出力している状態 -->

### 成功の目安

60秒ごとに価格とシグナルがログに流れれば成功。「HOLD」が続いても正常動作。

### もしうまくいかない場合

- **ボットが即座に終了する**：`while True`の前後でインデントが崩れていないか確認
- **価格が毎回`None`になる**：`get_btc_price()`単体で動くか先に確認する
- **Ctrl+Cで止まらない**：`KeyboardInterrupt`を`try/except`で拾っていない場合は少し待つか強制終了

---

## よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `ModuleNotFoundError: No module named 'requests'` | ライブラリ未インストール | `pip3 install requests`を再実行 |
| `KeyError: 'last'` | APIレスポンスの構造変更 | `print(response.json())`でキー名を確認 |
| `JSONDecodeError` | ネットワーク不安定でHTMLが返ってきた | `print(response.text)`で内容確認 |
| APIキーエラー（401） | キーの貼り間違い、または権限不足 | 取引所の管理画面でAPIキーの権限設定を確認 |
| 価格が古い | `timeout`値が短すぎる | `timeout=30`に延ばしてみる |

---

## 次に学ぶべきこと

この記事で価格取得とシグナル判定が動いたら、次のステップ：

1. **実際の注文APIを叩く**：認証付きリクエスト（HMAC-SHA256署名）の実装。C言語でハッシュ関数を扱ったことがあれば概念は掴みやすい。
   推奨リソース：各取引所の公式APIドキュメント（Coincheck、bitFlyer）

2. **バックテストを実装する**：過去の価格データでルールを検証する。「このロジックが過去1年で機能したか」を確認せずに実運用するのはギャンブルと変わらない。
   推奨リソース：`pandas`ライブラリの基礎、または書籍『Pythonによるファイナンス』

3. **リスク管理の実装**：損切りライン、ポジションサイズの計算、最大損失の上限設定。ここが一番地味で、一番重要。
   推奨リソース：「ケリー基準」「ドローダウン」で検索して概念を掴む

1ヶ月かけて3ステップ進めば、自分のロジックで動くボットの基礎は固まる。

---

## 関連リソース

この記事で扱った暗号資産取引を本格的に始めるなら：

- **Coincheck** [PR]
  国内最大級の取引所。APIドキュメントが日本語で整備されており、ボット開発の最初の練習台として使いやすい。
  → [詳細](https://coincheck.com/)

- **bitFlyer** [PR]
  APIの安定性と取引量で定評がある。Lightning APIはドキュメントが充実しており、本格的なボット開発に向いている。
  → [詳細](https://bitflyer.com/ja-jp/)

- **GMOコイン** [PR]
  手数料体系が明確で、少額から始めやすい。APIも整備されている。
  → [詳細](https://coin.z.com/jp/)

---

## 慣れてきた人へ

ボットの構造が掴めてきたら、取引所APIとの連携部分を体系的に実装したくなる場面が出てくる。

- **AutoTrader 実装学習キット**：FastAPI × React Native で作る外部API連携アプリの実装学習キット。ボットのバックエンドをAPIサーバとして構成したい段階で参考になる。
  （まだボットを動かし始めたばかりなら、先にステップ2・3を固めてから検討を）

---

**著者**：ぽん（@pon_freelance）
C言語実務15年、組み込み／制御系。
副業で技術記事販売と自作ツール販売をやっている。

書いているもの：
- AutoTrader 実装学習キット - FastAPI × React Native で作る外部 API 連携アプリ実装学習キット
（その他：（なし））

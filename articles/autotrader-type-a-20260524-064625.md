---
title: "Untitled"
emoji: "🔧"
type: "tech"
topics:
  - "Python"
  - "FastAPI"
  - "個人開発"
published: true
---
```markdown
---
title: "Zaif APIのorder_id=0問題を正しく扱う実装パターン"
platform: zenn
tags: [zaif, api, python, 仮想通貨, 自動売買]
---

# Zaif APIのorder_id=0問題を正しく扱う実装パターン

Zaif APIで注文を出したとき、レスポンスの`order_id`が`0`で返ってくることがある。これを「注文失敗」と誤判定して処理を止めた経験がある人向けに書く。自動売買ツールを実装していて、この挙動にハマった人が読むと役に立つはずだ。

---

## 結論から言う

Zaifの`order_id=0`は「注文失敗」ではなく「成行注文が即時約定した」ことを意味するケースがある。`order_id`の値だけで成否を判断する実装は壊れる。正しくは「HTTPステータスコード＋`order_id`＋残高変化」の3点セットで判定する必要がある。

---

## なぜこれが問題になるか

### Zaif APIの仕様上の問題

Zaif APIのトレードエンドポイント（`/1/trade`）は、注文が正常に受け付けられた場合に`order_id`を返す。通常の指値注文なら`order_id`は正の整数になる。ところが成行注文（`order_type: market`）を出したとき、板に十分な流動性があって即座に約定すると、`order_id`が`0`で返ってくることがある。

これはZaifの公式ドキュメントに明記されているわけではない。少なくとも2024年時点の実装では、成行注文の即時約定時に`order_id=0`が返るという挙動が確認されている。ドキュメントの記述が薄いため、実際に動かしてみるまでわからないというのが正直なところだ。

```python
# 成行注文のレスポンス例（即時約定時）
{
    "success": 1,
    "return": {
        "order_id": 0,        # ← これが問題のフィールド
        "received": "0.01",
        "remains": "0.0",
        "funds": {
            "jpy": 99500,
            "btc": 0.01
        }
    }
}
```

`order_id=0`を見た瞬間に「注文IDが無効」と判断してエラー処理に流すコードを書いていると、実際には約定しているのに「注文失敗」として再注文をかけてしまう。最悪の場合、二重注文になる。

### 自分が踏んだ失敗

自動売買ツールを作り始めた最初の3週間、成行注文の成否判定を`order_id > 0`で書いていた。テスト環境では指値注文しか使っていなかったので問題が出なかった。本番で成行注文を使い始めた初日に、残高確認で「あれ、BTC増えてる？」となって気づいた。ログを見ると「注文失敗→再注文」のサイクルが3回回っていた。幸い損失は1,500円程度で済んだが、流動性が高い時間帯に動いていたらもっと大きな額になっていた可能性はある。

---

## 問題の深掘り

### order_idが0になる条件

実装を繰り返して確認できた範囲では、以下の条件が重なると`order_id=0`が返る。

1. `order_type`が`market`（成行注文）
2. 注文数量に対して板の流動性が十分にある
3. 注文が即時約定する

逆に言うと、成行注文でも板が薄くて部分約定になる場合は、残った注文に対して正の`order_id`が振られることがある。`order_id=0`は「完全即時約定した成行注文」の識別子として機能しているとも解釈できる。

### 他の取引所との比較

bitFlyerやGMOコインでは、成行注文の即時約定でも正の`order_id`が返る。Zaifのこの挙動はかなり特殊で、複数取引所対応のコードを書くときに共通の判定ロジックを使うと必ずハマる。

取引所ごとに「注文成功の定義」が微妙にずれていることを、自分はZaifで初めて意識した。

### `success`フィールドを見ればいいのでは？

`success: 1`が返っていれば注文は受け付けられている、という判断は半分正しい。ただし`success: 1`かつ`order_id: 0`のケースが存在するため、「`success: 1`なら処理続行」だけでは後続の注文管理（オープンオーダーの監視、キャンセル処理）が壊れる。

`order_id`を使ってオープンオーダーを管理しているコードは、`order_id=0`を受け取った瞬間に「管理対象の注文ID」として登録しようとして、以降の`get_active_orders`で該当IDが見つからずにエラーループに入る。

---

## 解決手順

### ステップ1：注文レスポンスの判定ロジックを作り直す

まず「注文成功」の判定条件を整理する。

```python
def is_order_successful(response: dict) -> bool:
    """
    Zaif APIの注文レスポンスから成否を判定する。
    
    order_id=0は成行注文の即時約定を意味するため、
    successフィールドを優先して判定する。
    """
    if response.get("success") != 1:
        return False
    
    ret = response.get("return", {})
    
    # order_id=0は即時約定を意味するため、エラーではない
    order_id = ret.get("order_id", -1)
    if order_id == -1:
        # order_idフィールド自体が存在しない場合は異常
        return False
    
    return True
```

`order_id=0`を明示的に「有効なケース」として扱うことがポイントだ。`-1`や`None`と区別するために、デフォルト値を`-1`にしている。

### ステップ2：注文種別ごとに後続処理を分岐させる

`order_id=0`の場合は「即時約定済み」なので、オープンオーダー監視のキューに積む必要がない。

```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional

class OrderStatus(Enum):
    OPEN = "open"           # 未約定（指値等）
    FILLED = "filled"       # 即時約定（成行）
    FAILED = "failed"       # 注文失敗

@dataclass
class OrderResult:
    status: OrderStatus
    order_id: Optional[int]
    received: float
    remains: float
    funds: dict

def parse_order_response(response: dict) -> OrderResult:
    """
    Zaif APIの注文レスポンスをパースして、
    後続処理が使いやすい形に変換する。
    """
    if not is_order_successful(response):
        return OrderResult(
            status=OrderStatus.FAILED,
            order_id=None,
            received=0.0,
            remains=0.0,
            funds={}
        )
    
    ret = response["return"]
    order_id = ret.get("order_id", -1)
    received = float(ret.get("received", 0))
    remains = float(ret.get("remains", 0))
    
    # order_id=0かつremainsが0なら即時全量約定
    if order_id == 0 and remains == 0.0:
        status = OrderStatus.FILLED
    elif order_id > 0:
        status = OrderStatus.OPEN
    else:
        # order_id=0かつremainsが残っている場合（部分約定の可能性）
        # 実運用で確認できていないケースだが、念のため分岐
        status = OrderStatus.OPEN
    
    return OrderResult(
        status=status,
        order_id=order_id if order_id > 0 else None,
        received=received,
        remains=remains,
        funds=ret.get("funds", {})
    )
```

### ステップ3：オープンオーダー管理に組み込む

`OrderStatus.FILLED`の場合はキューに積まない。

```python
class OrderManager:
    def __init__(self):
        self.open_orders: dict[int, OrderResult] = {}
    
    def submit_order(self, response: dict) -> OrderResult:
        result = parse_order_response(response)
        
        if result.status == OrderStatus.FAILED:
            # ログ出力・アラート処理
            print(f"[ERROR] Order failed: {response}")
            return result
        
        if result.status == OrderStatus.OPEN and result.order_id is not None:
            # 未約定注文のみ監視キューに積む
            self.open_orders[result.order_id] = result
            print(f"[INFO] Open order registered: id={result.order_id}")
        
        if result.status == OrderStatus.FILLED:
            # 即時約定：残高変化を記録するだけでいい
            print(f"[INFO] Order filled immediately: received={result.received}")
        
        return result
```

### ステップ4：残高変化で約定を二重確認する

`order_id=0`の即時約定判定に不安がある場合は、注文前後の残高差分で確認する。

```python
def verify_fill_by_balance(
    balance_before: dict,
    balance_after: dict,
    currency_pair: str,  # 例: "btc_jpy"
    order_side: str      # "bid" or "ask"
) -> bool:
    """
    残高変化から約定を確認する。
    order_id=0の即時約定判定の補助として使う。
    """
    base, quote = currency_pair.split("_")  # "btc", "jpy"
    
    if order_side == "bid":  # 買い注文
        # base通貨（BTC等）が増えているはず
        diff = balance_after.get(base, 0) - balance_before.get(base, 0)
        return diff > 0
    else:  # ask: 売り注文
        # quote通貨（JPY等）が増えているはず
        diff = balance_after.get(quote, 0) - balance_before.get(quote, 0)
        return diff > 0
```

この残高確認は「APIレスポンスの`funds`フィールド」を使っても大体わかるが、通信エラーで`funds`が取得できなかった場合に備えて、別途`get_info2`エンドポイントで残高を取り直すほうが確実だ。

---

## 落とし穴と対処

### 落とし穴1：`remains`フィールドを信用しすぎる

`order_id=0`かつ`remains=0`で「全量約定」と判断するロジックを書いたが、`remains`が小数点以下の丸め誤差で`0.00000001`のような値になるケースがある。`remains == 0.0`の厳密比較は避けて、`remains < 0.00001`のような閾値判定にしておいたほうが安全だ。

```python
REMAINS_THRESHOLD = 1e-5  # 0.00001以下は約定済みとみなす

if order_id == 0 and remains < REMAINS_THRESHOLD:
    status = OrderStatus.FILLED
```

### 落とし穴2：エラーレスポンスでも`order_id=0`が返る

認証エラーや残高不足のエラー時にも、`order_id`フィールドが`0`で返ることがある（`success: 0`と一緒に）。`success`フィールドを先にチェックしていれば問題ないが、エラーハンドリングを後から追加したときに判定順序を逆にしてしまうと詰まる。判定の順序は「`success`→`order_id`」を固定しておく。

### 落とし穴3：Zaif APIのレートリミット

ZaifのプライベートAPIは1秒あたり数リクエストの制限がある。残高確認を二重でかけると、注文→残高確認→残高確認のシーケンスで制限に引っかかることがある。自分は`time.sleep(0.5)`を挟んで対処したが、高頻度取引では別途レートリミット管理が必要になる。

この手の「取引所固有のクセ」は、実際に動かさないと文書化されていない挙動が多い。自分がAutoTrader実装学習キットを作ったのも、Zaifのこの問題を含めた取引所固有の仕様差を、コードとして残しておきたかったからだ。

---

## 関連ツール

**[AutoTrader 実装学習キット](https://autotrader-lp.onrender.com/?utm_source=qiita&utm_medium=article&utm_campaign=autotrader)**（著者開発）

FastAPI × React Native で作る外部API連携アプリの実装学習キット。Zaifの`order_id=0`問題を含む取引所固有の仕様差を吸収する設計パターンと、Zaif/bitFlyer/GMO/Coincheck/Binance Japan の5取引所対応サンプルコードを収録している。スタンダード版（国内4社API連携サンプル）¥4,950から。

---

## まとめ

Zaifの`order_id=0`は「成行注文の即時約定」を意味するケースがある。`order_id > 0`を注文成功の条件にしているコードは、この挙動で誤動作する。`success`フィールドを優先して判定し、`order_id=0`を明示的に「即時約定済み」として扱う分岐を入れることで対処できる。残高変化による二重確認を組み合わせると、さらに堅牢になる。

---

**著者**：ぽん（@pon_freelance）
C言語実務15年、組み込み／制御系。
副業で技術記事販売と自作ツール販売をやっている。

書いているもの：
- AutoTrader 実装学習キット - FastAPI × React Native で作る外部 API 連携アプリ実装学習キット
（その他：（なし））
```
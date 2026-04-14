---
title: "Zaifのorder_id=0にハマった話 ── 即時約定の罠と3段階フォールバックで解決した全記録"
emoji: "🔧"
type: "tech"
topics:
  - "Python"
  - "FastAPI"
  - "個人開発"
published: true
---
## 問題の症状

仮想通貨の自動売買Botを運用していて、Zaifだけ奇妙な挙動が起きた。

**注文を出すと `order_id=0` が返ってくる。**

最初は「注文失敗か？」と思った。他の取引所（bitFlyer、GMOコイン、Coincheck）では注文IDとして `12345678` みたいな数値が返ってくるのが普通だ。0って何だ。

実際に起きた症状はこうだった：

- 注文を出すと `order_id=0` が返る
- Botが「注文失敗」と判断してリトライする
- 毎分リトライが走って**同じ注文が無限に繰り返される**
- 気づいたときには同一通貨ペアで重複ポジションが大量発生

```python
# 当初のコード（バグあり）
if not order_id:  # order_id="0" は truthy なのでここを通過しない
    logger.error("注文失敗")
    return

# order_id="0" はここに来てしまい、約定確認ループに入る
await check_fill(order_id)  # 永遠に約定確認できない → リトライ
```

`order_id="0"` は文字列としては truthy（Pythonでは `bool("0") == True`）なので、失敗判定をすり抜ける。そしてこのIDで約定確認しても当然ヒットしない。結果、無限リトライに陥る。

## 原因調査

### Zaif API仕様の罠

調べてみると、Zaifの公式API仕様にこう書いてあった。

> 注文が即座に全量約定した場合、`order_id` は `0` を返します。板に注文が残らないため、注文IDは発行されません。

つまり **`order_id=0` は失敗ではなく「即座に全部約定した」という成功レスポンス** だった。

他の取引所は即時約定でも注文IDを返すが、Zaifは「板に残らない＝IDなし」というポリシーらしい。ccxt（暗号資産取引所の統一APIライブラリ）にもこの挙動は明記されていなかった。

### レスポンスの中身を見る

Zaifのレスポンスにはこんなフィールドが入っている：

```json
{
  "success": 1,
  "return": {
    "received": 0.001,
    "remains": 0,
    "order_id": 0
  }
}
```

- `received`: 約定した数量
- `remains`: 未約定の残り（0 = 全量約定）
- `order_id`: 0 = 即時約定

**`received` と `remains` を見れば約定したかどうか分かる。** order_idだけで判断していたのが根本原因だった。

### もう一つの罠：重複注文

さらに厄介だったのが、注文の重複防止ロジック。

```python
# 修正前：注文実行「後」にフラグを立てていた
result = await place_order(signal)
bot._order_pending = True  # ← ここでセット

# 問題：place_order()の実行中（数秒）に次のサイクルが来ると
# フラグが立っていないので同じ注文を重複実行してしまう
```

Zaifの板追随注文では最大40秒の追跡時間がある。この間に次の売買サイクルが来ると、ペンディングフラグが立っていないので同じ注文が走る。マルチBot環境だと残高変動も重なって、カオスになる。

## 解決策コード

最終的に3つの修正で解決した。

### 修正1：order_id=0の正しいハンドリング

```python
# order_id=0: Zaif公式仕様 = 即時全量約定
if not order_id or order_id == "0":
    zaif_return = (order_result.get("info") or {}).get("return", {})
    received = zaif_return.get("received", 0)
    remains = zaif_return.get("remains", 0)

    if received and float(received) > 0:
        # 即時約定確定（公式仕様に基づく判定）
        filled_amount = float(received)
        order_status = "FILLED"
        order_id = f"zaif_instant_{datetime.now(timezone.utc).strftime('%H%M%S')}"
        logger.info(f"Zaif即時約定: received={received} remains={remains}")
    else:
        # received=0 → 本当に失敗（残高不足など）
        logger.warning(f"注文失敗（order_id=0, received=0）")
        return
```

ポイントは **order_idではなく `received` フィールドで約定を判定する** こと。order_id=0でもreceived > 0なら成功。

### 修正2：ペンディングフラグを注文「前」に移動

```python
# 修正後：注文実行「前」にフラグを立てる
bot._order_pending = True
bot._order_pending_since = datetime.now(timezone.utc)

result = await place_order(signal)

# 10分経過したら自動解除（デッドロック防止）
if (datetime.now(timezone.utc) - bot._order_pending_since).total_seconds() > 600:
    bot._order_pending = False
```

これで注文実行中に次のサイクルが来ても、フラグが立っているので重複注文を防げる。

### 修正3：3段階フォールバック約定確認

Zaifは約定確認APIの挙動も独特なので、3段階のフォールバック戦略を組んだ。

```python
async def check_fill(order_id: str, exchange: str) -> str:
    # Stage 1: アクティブ注文一覧から確認
    open_orders = await fetch_open_orders()
    if order_id not in [o["id"] for o in open_orders]:
        return "FILLED"  # 板から消えた = 約定

    # Stage 2: 約定履歴から確認（Zaifはタイムウィンドウ照合）
    if exchange == "zaif":
        closed = await fetch_closed_orders(since=order_time)
        # order_idではなく時刻・数量・価格でマッチング
        matched = find_matching_order(closed, signal)
        if matched:
            return "FILLED"
    else:
        trades = await fetch_my_trades()
        if any(t["order"] == order_id for t in trades):
            return "FILLED"

    # Stage 3: 残高差分で最終確認（最終手段）
    balance_diff = current_balance - pre_order_balance
    if abs(balance_diff) > threshold:
        return "FILLED"

    return "PENDING"
```

Stage 2でZaifだけ特別扱いしているのは、`fetchMyTrades` が使えないため。`fetchClosedOrders` で時刻ウィンドウ照合するしかない。

## まとめ

Zaifの `order_id=0` 問題は、一見シンプルに見えて3つのバグが絡み合っていた。

| 問題 | 原因 | 修正 |
|------|------|------|
| 即時約定を失敗と誤判定 | order_idだけで判定 | `received`フィールドで判定 |
| 同じ注文が無限リトライ | ペンディングフラグのタイミング | 注文前にフラグセット + 10分自動解除 |
| 約定確認がヒットしない | Zaif固有のAPI制約 | 3段階フォールバック確認 |

教訓は「**取引所ごとにAPIの挙動は全然違う**」ということ。ccxtで抽象化されていても、エッジケースは取引所固有のレスポンスを自分で確認するしかない。

特にZaifはorder_id=0以外にも独自仕様が多い。自動売買Botを作るなら、本番投入前にサンドボックス環境で全パターンをテストすることを強くおすすめする。

---

自動売買で取引回数が増えると確定申告が地獄になる。取引履歴をアップロードするだけで損益計算を自動化してくれるツールを使っておくのがおすすめ → [利用者No1の仮想通貨税金計算サービス【CRYPTACT（クリプタクト）】](https://px.a8.net/svt/ejp?a8mat=4B1EPO+16V8C2+4DGW+5YJRM)

---
title: "bitFlyer API制限を超えないトークンバケット設計と、運用で崩れやすい5つの点"
emoji: "🔧"
type: "tech"
topics:
  - "Python"
  - "FastAPI"
  - "個人開発"
published: true
---
bitFlyerのAPIで自動売買や取引記録ツールを組んでいる人向けの記事。リクエスト制限に当たってしまう人、当たるのが怖くて固定のsleepを入れ、思ったほど速度が出ていない人が対象。

トークンバケット自体の説明は他にも多い。この記事は、バケットを書いたあとに崩れやすい箇所を中心に整理する。以下の失敗パターンは自分の障害報告ではない。設計の構造から起きうるものを、起きる順に並べたものである。

## 結論

制限を守る仕組みはトークンバケットで足りる。ただし実装の難所はバケット本体より周辺にある。

- 制限値はコードに埋め込まず、設定として外に出す
- 注文系と参照系でバケットを分ける
- 待ち時間に上限を付け、古い注文を出さない
- 429を受けたらバケットを意図的に空にして、リトライで悪化させない
- バケットはプロセスをまたいで共有する

この5点を押さえておけば、本番に載せたあとの手戻りがかなり減るという実感がある。

## 問題の深掘り：固定sleepでは足りない理由

最初に思いつくのは、リクエストの間に固定時間のsleepを入れる方法だ。制限が1分あたりN回なら、60÷N秒ずつ待てば超えない。動くことは動くが、運用に載せると3つの不都合が出る。

1つ目は、速度の無駄が大きいこと。取引所側は一定の窓の中で回数を数えるので、窓に余裕があるなら短時間に固めて送ってもよい。固定sleepはこの余裕を使えない。注文を出したい瞬間に、参照系の待ち時間に引っかかることもある。

2つ目は、数えている主体が自分だけではないこと。同じAPIキーやIPを、約定確認のポーリング、残高取得、注文送信という複数のコンポーネントが使う。それぞれが自分のsleepだけを守っても、合算した回数は誰も見ていない。

3つ目は、取引所側の窓と自分の時計がずれること。自分では1分間にN回以内のつもりでも、窓の切れ目をまたぐとバーストとして数えられる場合がある。

要するに、リクエストを送る側で「全体の回数」を一箇所で管理する必要がある。ここまでで、固定sleepは出発点にはなっても終点にはならないとわかる。

bitFlyerを含む取引所APIのこの手の実装例は、あまり多く出回っていない。自分は外部API連携の実装を学習用にまとめた[AutoTrader 実装学習キット](https://autotrader.ponfreelance.com/sales/?utm_source=zenn&utm_medium=article&utm_campaign=autotrader&utm_content=affiliflow)を作っていて、bitFlyerを含む国内取引所を題材にしている。この記事はその前段にあたる設計の話として読んでもらえればいい。

## 解決手順

### ステップ1：制限値を設定に出す

bitFlyerのリクエスト制限は、APIの種類（公開・認証が必要なもの）や、注文系の追加条件で扱いが分かれる。値は改定されることもあるので、コードに数字を埋め込まない。公式ドキュメントで現行値を確認し、設定ファイルか環境変数から読む。

設定として持つのは、バケット容量（バースト許容量）、1秒あたりの補充レート、待ちの上限秒数の3つ。補充レートは公式の上限ぎりぎりではなく、1〜2割下げて始める。これは自分の時計と取引所側の窓がずれる分の余裕である。

### ステップ2：用途別にバケットを分ける

注文の送信と取消は、遅れると約定の機会や損失に直結する。残高や約定履歴の参照は、数秒遅れても致命的ではない。同じバケットに入れると、参照系のポーリングが注文を押しのけうる。

分け方は2つ作るだけでいい。注文系のバケットは専用にして、参照系より優先する。参照系は残りの枠を使う。ただし取引所側が合算で数える制限もあるので、2つのバケットの補充レートの合計が公式の上限を超えないように設定する。

### ステップ3：待ちに上限を付ける

トークンが無ければ待つ、というだけの実装は危うい。待っている間に相場が動き、古い価格の注文が遅れて出てしまう。acquireにタイムアウトを持たせ、超えたら例外を投げて呼び出し側に判断を戻す。注文系は短く、参照系は長めに設定する。

### ステップ4：429を受けたときの振る舞い

制限超過を示す応答が来たら、それは自分のバケットの見積もりがずれていたという合図だ。このとき即座にリトライすると、そのリトライがまたトークンを消費し、悪循環になりやすい。

対処は2段構え。まずバケットのトークンを0にして、一定時間は補充も止める。次に、その間のリクエストは待たせるか、タイムアウトで落とす。再開後は補充レートを一時的に下げ、問題なく通る回数が続いたら元に戻す。

### ステップ5：バケットの置き場所

バケットをアプリ内の変数として持つと、そのプロセスの中でしか効かない。ワーカーを複数立てる構成や、別プロセスのバッチが同じAPIキーを使う構成では、バケットが分裂する。次節で扱う。

## コード例

Pythonでの最小実装を示す。FastAPIのようなasync環境を想定している。

```python
import asyncio
import time


class RateLimitWaitTimeout(Exception):
    pass


class TokenBucket:
    def __init__(self, capacity: float, refill_per_sec: float):
        self.capacity = capacity
        self.refill_per_sec = refill_per_sec
        self.tokens = capacity
        self.updated = time.monotonic()
        self.blocked_until = 0.0
        self._lock = asyncio.Lock()

    def _refill(self, now: float) -> None:
        if now < self.blocked_until:
            self.updated = now
            return
        elapsed = now - self.updated
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_per_sec)
        self.updated = now

    async def acquire(self, timeout: float) -> None:
        deadline = time.monotonic() + timeout
        while True:
            async with self._lock:
                now = time.monotonic()
                self._refill(now)
                if now >= self.blocked_until and self.tokens >= 1:
                    self.tokens -= 1
                    return
                wait = max(self.blocked_until - now, (1 - self.tokens) / self.refill_per_sec)
            if time.monotonic() + wait > deadline:
                raise RateLimitWaitTimeout()
            await asyncio.sleep(wait)

    async def penalize(self, seconds: float) -> None:
        async with self._lock:
            self.tokens = 0
            self.blocked_until = time.monotonic() + seconds


# 値は公式ドキュメントで現行値を確認して設定する
order_bucket = TokenBucket(capacity=5, refill_per_sec=2.0)
query_bucket = TokenBucket(capacity=10, refill_per_sec=3.0)
```

時刻にはtime.timeではなくtime.monotonicを使っている。システム時刻の補正で時計が飛んでも、トークンが急に増えたり減ったりしないためだ。

呼び出し側は次の形になる。

```python
async def place_order(client, params):
    await order_bucket.acquire(timeout=2.0)
    resp = await client.post("/v1/me/sendchildorder", json=params)
    if resp.status_code == 429:
        await order_bucket.penalize(seconds=30)
        raise RuntimeError("rate limited")
    return resp
```

## 落とし穴と対処

実際の障害ではなく、構造から起きうる失敗として整理する。レビューのときに見る観点としても使える。

1つ目は、ワーカー数を増やした瞬間にバケットが分裂すること。uvicornを複数ワーカーで動かすと、各プロセスが別のバケットを持つ。ワーカー2つなら実効レートは2倍になる。対処は、バケットをRedisなどの共有ストアに置くか、APIを叩く役割を1プロセスに集約すること。後者のほうが単純で壊れにくい。

2つ目は、再起動直後の満タン開始。バケットは容量いっぱいから始まるので、異常終了とリスタートが繰り返されると、そのたびにバーストが出る。起動直後は容量の半分程度から始めるか、短いウォームアップ時間を設ける。

3つ目は、認証キーを分けたつもりで制限が分かれていないこと。制限がキー単位かIP単位かは、APIの種類によって異なる場合がある。IP単位の制限があるなら、キーごとにバケットを作っても同じIPから送る限り合算で数えられる。公式の記述を確認し、単位に合わせてバケットを持つ。

4つ目は、待ち時間の上限なし。前述のとおり、古い注文が遅れて出る。さらに、待っているタスクが溜まると、制限が解けた瞬間に一斉に送られて、また制限に当たる。タイムアウトで落とす設計にしておけば、溜まったタスクが詰まることはない。

5つ目は、429の原因を呼び出し側で握りつぶすこと。リトライだけして成功すれば問題が見えなくなり、バケットの設定が甘いまま運用が続く。429を受けた回数はログと指標に残し、ゼロでなければ補充レートを下げる材料にする。

## 関連ツール

外部API連携の実装を、FastAPIとReact Nativeの構成で学べるキットを作っている。取引所APIごとのクセの吸収や、異常検知・安全装置の実装例を含む。

[AutoTrader 実装学習キット](https://autotrader.ponfreelance.com/sales/?utm_source=zenn&utm_medium=article&utm_campaign=autotrader&utm_content=affiliflow)

## まとめ

レート制限の対策は、トークンバケットを書いて終わりではなく、そのまわりの設計で決まる。

制限値は設定に外出しして、用途別にバケットを分ける。待ちには上限を付け、429を受けたらバケットを空にする。複数プロセスで使うなら、バケットの置き場所を最初に決めておく。

この5点を書き出してから実装に入ると、運用に載せたあとの手戻りが少なくなるはずだ。

---

**著者**：ぽん（@pon_freelance）
C言語実務23年、組み込み／制御系。
副業で技術記事販売と自作ツール販売をやっている。

書いているもの：
- AutoTrader 実装学習キット - FastAPI × React Native で作る外部 API 連携アプリ実装学習キット
（その他：（なし））

**開発の裏側を購読できます** — AutoTrader のリリースごとに「何を・なぜ・どう変えたか」を 2,000〜4,000 字で書き残しています。バグの原因、取引所 API 変更への追従、設計判断のトレードオフまで。
→ [AutoTrader開発ログ（月500円・いつでも解約可）](https://note.com/clab_jp/membership)

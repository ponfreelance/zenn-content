---
title: "Render Free + Groq + Cookie 署名で AI SaaS を ¥0 で立ち上げた話"
emoji: "💰"
type: "tech"
topics:
  - "fastapi"
  - "python"
  - "個人開発"
  - "saas"
published: true
---

自分は C 言語を 23 年使ってきたフリーランスのエンジニアで、最近は Python + FastAPI で個人開発の SaaS を回しています。期限切れ特許に AI で「商品化スコア」を付けて中国輸入セラー向けに提示する、というニッチな SaaS を立ち上げました。本稿はその SaaS を **Render Free + Groq 無料枠 + 署名 Cookie** で **月 ¥0 のまま運用** するための実装メモです。

## この記事の前提（SCQA）

- **状況（S）**: 個人開発で AI SaaS をマネタイズしたいが、AWS Lambda も含めてインフラ代と LLM コストが見えない。最初の有料客が付くまではキャッシュアウトを 0 円に貼り付けたい。
- **問題（C）**: Render Free は 15 分アクセスがないとスリープするし、SQLite は再起動で揮発する。Claude API は単価が読めず、悪意のリクエストで請求が爆発する怖さがある。
- **問い（Q）**: 「常時起動 + 顧客の Pro 状態が消えない + LLM コストが完全上限」を **¥0 のまま** どう成立させるか？
- **答え（A）**: ① UptimeRobot で 5 分 ping、② itsdangerous の署名 Cookie に顧客状態を埋める、③ Free は Groq 固定・Pro は月間上限で API を物理的にカットする、④ 黒字後は Render Starter $7/月 へ昇格する 4 段階。

## 1. ¥0 構成の全体像

```
Browser ─Cookie{u,plan,evals,pro_until}→ FastAPI(Render Free)
                                          ├─ SQLite(揮発)
                                          ├─ Groq Llama-3.3-70B(無料枠)
                                          └─ Claude Sonnet(売上後追加)
       UptimeRobot ─5min HTTP ping→ /lp
```

| 部品 | 役割 | 月コスト |
| --- | --- | --- |
| Render Free | FastAPI ホスト | ¥0 |
| Groq | LLM（1 次評価） | ¥0（無料枠） |
| UptimeRobot | スリープ防止の 5 分 ping | ¥0 |
| SQLite | DB（永続ディスクなしで揮発） | ¥0 |
| Claude（任意） | LLM（2 次評価） | 売上が立ってから |

## 2. 署名 Cookie で「DB 揮発しても顧客 Pro が消えない」を作る

Render Free は SQLite が揮発する。普通に作ると **Pro 課金した顧客の状態がインスタンス再起動で消える**。これは売上漏れに直結する致命傷だが、署名 Cookie に顧客状態を埋め込めば回避できる。

itsdangerous の `URLSafeSerializer` を使い、SECRET_KEY で署名した辞書を Cookie に焼く。改ざんされたら署名検証で落ちる。

```python
from itsdangerous import URLSafeSerializer, BadSignature

SESSION_COOKIE = "tps_session"
signer = URLSafeSerializer(SESSION_SECRET, salt="tps")

def _read_state(request):
    raw = request.cookies.get(SESSION_COOKIE)
    if not raw:
        return _new_state()
    try:
        data = signer.loads(raw)
        return {
            "u": str(data.get("u")),
            "plan": str(data.get("plan", "free")),
            "evals": int(data.get("evals", 0)),
            "pro_until": data.get("pro_until"),
        }
    except BadSignature:
        return _new_state()
```

Cookie に持たせる中身は最小:

- `u`: 匿名ユーザー識別子（再訪時の名寄せ用）
- `plan`: `free` か `pro`
- `evals`: 累計評価回数（無料枠 5 件のカウント）
- `pro_until`: Pro 有効期限の epoch 秒（30 日）

期限切れの Pro はミドルウェアで自動 `free` に降格させる。これをやらないと「Cookie の改ざんはできないが、永遠に Pro になれる」という穴ができる。

```python
@app.middleware("http")
async def session_middleware(request, call_next):
    state = _read_state(request)
    if state["plan"] == "pro" and state.get("pro_until"):
        if int(state["pro_until"]) < int(time.time()):
            state["plan"] = "free"
            state["pro_until"] = None
    request.state.session = state
    response = await call_next(request)
    _write_state(response, state)
    return response
```

これで「**サーバ DB が揮発しても、顧客のブラウザ Cookie の中にしか有効期限が無いので、Render Free でも売上が消えない**」が成立する。

実物の挙動を確かめたい場合は [この LP](https://cutt.ly/StXzwsWV) で `Free → /login → Pro` を 1 周してから DevTools の Cookie タブを覗くと中身が見える。

## 3. LLM コストを「物理的に」カットする 2 層

「悪意のリクエストで Claude API が爆発する」を防ぐには、**プラン別に呼ぶプロバイダ自体を変える** のが最も確実。Free=Groq 固定（無料枠）、Pro=Claude（月額上限を別途強制）にする。

```python
def evaluate_patent(title, abstract, claims, provider=None):
    actual = (provider or os.environ.get("LLM_PROVIDER", "claude")).lower()
    prompt = _build_prompt(title, abstract, claims)
    if actual == "groq":
        raw = _evaluate_groq(prompt)
    elif actual == "mock":
        raw = _evaluate_mock(prompt, content=...)
    else:
        raw = _evaluate_claude(prompt)
    return _normalize(raw)
```

- **Free**: `provider="groq"` を強制。月額 ¥0 で課金リスク 0。
- **Pro**: ハイブリッド。Groq で 1 次評価し、`score >= 7` の特許のみ Claude で 2 次評価で上書き。Pro 月間上限 `PRO_MONTHLY_LIMIT=500` を超えたら Claude 呼び出しを止める。

`_normalize` で LLM 出力を必ず固定スキーマに矯正している。LLM が JSON で返さなくても、フィールドが欠けても落ちない。スキーマの defensive normalization は個人開発 SaaS で軽視されがちだが、ここを手抜きすると本番で壊れて顧客が逃げる。組み込みでヌルチェック漏れを長く見てきた人間としては、この層は省けない。

## 4. UptimeRobot で寝かせない（最小コストで）

Render Free の 15 分スリープ問題。UptimeRobot の Free プランは 50 モニタまで無料、5 分間隔 HTTP ping が打てる。

| 項目 | 値 |
| --- | --- |
| Monitor Type | HTTP(s) |
| URL | `https://<service>.onrender.com/lp` |
| Interval | 5 minutes |

これで月 ¥0 のまま常時起動が成立する。LP に Cookie 発行を含めると毎回 304 にならず 200 になるので、Render Free のヒット判定にも乗りやすい。

## 5. 売上が立った後の昇格パス

ここまでの構成で **有料客が付く前は完全に ¥0 のキャッシュアウト** が成立する。1 人有料客が出たら次の昇格を検討する。

| 売上ステージ | やること | 月コスト |
| --- | --- | --- |
| 0 円 | Render Free + Groq + UptimeRobot | ¥0 |
| 1 人 〜 | Render Starter $7/月 へ昇格、永続ディスク追加 | ¥1,100 |
| 5 人 〜 | Stripe Webhook 自動化、Claude 増額 | ¥3,000-¥5,000 |
| 10 人 〜 | Postgres 移行、独自ドメイン | ¥5,000-¥10,000 |

基本姿勢は「売上が立つまでサーバ代を払わない」「売上に応じて段階的に昇格」の 2 つだけ。個人開発を続けてると、失敗しても財布が痛まない構成にしておくのが結局長続きする。

## まとめ

- Render Free + Groq + 署名 Cookie で個人開発 AI SaaS は **¥0 で立ち上げられる**
- 署名 Cookie に顧客状態を埋めると、サーバ DB が揮発しても売上が消えない
- LLM コストはプロバイダ別出し分けで物理的にカット
- UptimeRobot 5 分 ping で常時起動を成立させる
- 売上が立ってから段階的に昇格する

実物の SaaS を **無料 5 件試せます** → [期限切れ特許 × AI で商品ネタを発掘する SaaS](https://cutt.ly/StXzwsWV)（中国輸入セラー / D2C 向け）。中国輸入セラー向けの完全ガイドを Brain にまとめました → [**期限切れ特許 × AI で中国輸入の差別化ネタを発掘する完全ガイド**（Brain ¥4,980）](https://cutt.ly/9tXUfDwn)

---

著者: ぽん（C 言語 23 年 / フリーランス組み込みエンジニア）
X: [@pon_freelance](https://x.com/pon_freelance)

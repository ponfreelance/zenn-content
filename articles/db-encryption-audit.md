---
title: "本番DBの暗号資産APIキー、ちゃんと暗号化されてるか検証した話"
emoji: "🔐"
type: "tech"
topics:
  - "Python"
  - "PostgreSQL"
  - "セキュリティ"
  - "暗号資産"
published: true
---

## はじめに

暗号資産の自動売買BOTを個人開発して販売しています。購入者の取引所APIキー（Zaif / bitFlyer / Coincheck / Binance Japan）がPostgreSQLに保存されるのですが、ある日ふと思いました。

**「このAPIキー、本当に暗号化されてDBに入ってる？コードでは暗号化してるつもりだけど、実DBは確認したことないな」**

コードレビューでは「`encrypt()` 呼んでるからOK」で済ませがちですが、実際のDBに入っている値が暗号文かどうかは、DBに繋いで確認しない限りわかりません。マイグレーション事故や環境変数の設定漏れで「暗号化コードはあるが実際は平文」という状態は十分ありえます。

そこで、本番DBに対して読み取り専用の監査を実施しました。この記事ではその方法と結果を共有します。

## 前提

- バックエンド: FastAPI + SQLAlchemy 2.0
- DB: Railway PostgreSQL
- 暗号化: `cryptography` パッケージの Fernet（AES-128-CBC + HMAC-SHA256）
- 対象テーブル: `exchange_credentials`（取引所APIキー）、`ai_credentials`（AIプロバイダーAPIキー）

## 監査の全体像

```
PHASE 1: コード調査（grep のみ）
  → モデル定義・暗号化ユーティリティ・呼び出し箇所を特定

PHASE 2: DB 直接確認（SELECT のみ）
  → テーブル存在確認・カラム型・暗号文プレフィックス確認

PHASE 3: 暗号化キーの保管確認
  → 環境変数管理か？Git に混入していないか？

PHASE 4: ログ漏洩確認
  → logger/print に平文キーが流れていないか
```

**全フェーズ読み取り専用**。本番DBへの書き込みは一切行いません。

## PHASE 1: コード調査

### モデル定義を確認

```bash
grep -rn "api_key\|api_secret\|exchange_credential" backend/models.py
```

```python
class ExchangeCredential(Base):
    __tablename__ = "exchange_credentials"
    exchange: Mapped[str] = mapped_column(String, primary_key=True)
    api_key_enc: Mapped[str] = mapped_column(String, nullable=False)
    secret_enc: Mapped[str] = mapped_column(String, nullable=False)
```

カラム名が `api_key` ではなく **`api_key_enc`**。設計段階から暗号化前提の命名です。

### 暗号化ユーティリティ

```python
# backend/crypto.py
from cryptography.fernet import Fernet

def _get_fernet() -> Fernet:
    raw_key = os.getenv("ENCRYPTION_KEY")
    if not raw_key:
        raise RuntimeError("ENCRYPTION_KEY 環境変数が設定されていません")
    key_bytes = hashlib.sha256(raw_key.encode()).digest()
    return Fernet(base64.urlsafe_b64encode(key_bytes))

def encrypt(plaintext: str) -> str:
    return _get_fernet().encrypt(plaintext.encode()).decode()

def decrypt(ciphertext: str) -> str:
    return _get_fernet().decrypt(ciphertext.encode()).decode()
```

### 書き込み・読み出し箇所の網羅確認

書き込みは `routers/exchanges.py` と `routers/ai_credentials.py` の create/update — **4箇所のみ**、すべて `encrypt()` 経由。

読み出しは `scheduler.py` / `routers/bots.py` 等で、すべて ccxt クライアント生成に直接渡されるだけ。APIレスポンスやログには流れません。

## PHASE 2: DB 直接確認

### 監査スクリプト

```python
conn = psycopg2.connect(DATABASE_URL, connect_timeout=15)
conn.set_session(readonly=True, autocommit=True)  # 読み取り専用を強制
cur = conn.cursor()

# 先頭8文字だけ取得して暗号文か判定
cur.execute("""
    SELECT exchange, LEFT(api_key_enc, 8), LEFT(secret_enc, 8)
    FROM exchange_credentials
""")
```

`readonly=True` で読み取り専用セッションを強制。万一 `UPDATE` が紛れ込んでも実行されません。

### 結果

```
取引所             api_key_enc    secret_enc
--------------------------------------------------
Binance Japan     gAAAAABp        gAAAAABp
bitFlyer          gAAAAABp        gAAAAABp
Coincheck         gAAAAABp        gAAAAABp
Zaif              gAAAAABp        gAAAAABp
```

**全レコードが `gAAAAA` で始まっています。** Fernet トークンの標準プレフィックスです。

Fernet のトークン構造:
- `g` = base64エンコードされたバージョンバイト（`0x80`）
- `AAAAB` = タイムスタンプ先頭（base64）
- 以降: IV + 暗号文 + HMAC

## PHASE 3: 暗号化キーの検証

```bash
# .env が Git 追跡されていないか → .env.example のみ
git ls-files | grep ".env"

# Git 全履歴に .env が一度も追加されていないか → 空
git log --all --diff-filter=A -- '*.env'

# コード内ハードコード → なし
grep -rn "Fernet(b" backend/
```

最終確認として、本番キーで4件すべての復号を試行:

```
Binance Japan: OK
bitFlyer:      OK
Coincheck:     OK
Zaif:          OK
```

4/4 全件復号成功。復号した値は `del` で即時破棄し、画面には出力していません。

## PHASE 4: ログ漏洩確認

```bash
grep -rn "logger.*decrypt\|print.*decrypt" backend/
# → No matches
```

直接的な漏洩はゼロ。ただし `scheduler.py` で ccxt 例外を `logger.error(f"... {e}")` で出力する箇所があり、取引所エラーレスポンスにリクエスト断片が含まれる稀なケースでは理論上キー断片がログに残る可能性があります（低リスク）。

## 結果

| 観点 | 結果 |
|---|---|
| モデルに平文カラム | ❌ なし（`_enc` サフィックスのみ） |
| 全書き込みが `encrypt()` 経由 | ✅ |
| 実DBに暗号文が入っている | ✅ 全件 `gAAAAA` |
| 本番キーで復号できる | ✅ 4/4 |
| Git に暗号化キー混入 | ❌ なし |
| ログに平文キー | ❌ 直接的な漏洩なし |

## 学んだこと

1. **コードレビューだけでは不十分**。実DBの値を確認して初めて「暗号化されている」と言い切れる
2. **`readonly=True` セッションは必須**。監査スクリプトにうっかり `UPDATE` が入っても安全
3. **Fernet の `gAAAAA` プレフィックスは判定に使える**。先頭数文字で即座に暗号化済みか判断可能
4. **例外メッセージは盲点**。直接的なログ禁止でも、`{e}` 経由で漏れうる

暗号資産BOTを運用している個人開発者は、ぜひ一度「本番DBの値を直接確認する」監査をやってみてください。全て読み取り専用で、本番環境に一切影響を与えません。

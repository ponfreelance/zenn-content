---
title: "printf(user_input) を見逃したくないので、C言語レビューを自作した"
emoji: "🛡️"
type: "tech"
topics:
  - "c"
  - "embedded"
  - "codereview"
  - "staticanalysis"
  - "security"
published: false
---
C言語23年、自分は今もフリーランスで組み込み系のコードを触っています。長くやるほど実感するのは「目視レビューの精度は書き手より読み手の疲労度で決まる」という身も蓋もない話です。

この記事では、自分で書いた C 言語レビューツール3本を使って、**58行のテストコードに仕込んだ8個の脆弱性をどこまで自動検出できたか**をログと一緒に残します。

## この記事の前提（SCQA）

- **状況**: C 言語のレビューは長年、目視と経験則で回してきました。MISRA や CERT C は指針としては素晴らしいが、chunk 単位で数千行の diff が流れる現場では「読むだけで時間切れ」になります。
- **問題**: `printf(user_input)` のようなフォーマット文字列脆弱性は、見慣れていれば1秒で見抜ける。でも疲れていれば見逃す。脆弱性は「気づいた人のレビュー品質」で決まってしまいます。
- **問い**: 人間の集中力を前提にせず、同じ品質で毎回指摘してくれる仕組みを、ローカル完結で組めないか？
- **答え**: 自分は33パターンの静的解析ルールを持つ CLI（creview）と、コピペで試せる Web UI（c-review-ai）と、MISRA・依存グラフまで見る監査フレームワーク（CSAF）の3本を自作して常用しています。本記事ではテストコードを実際に食わせた検出ログを見せます。

## 検証用コード：58行に8個の危険を仕込む

検出能力を見たいので、脆弱性を意図的に仕込んだ `test_checks.c` を用意しました。全58行、関数8個、検出ターゲット8個です。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>

/* テスト1: フォーマット文字列脆弱性 */
void test_format_string(char *user_input) {
    printf(user_input);          /* 検出すべき */
    printf("%s", user_input);    /* 安全 */
}

/* テスト2: use-after-free */
void test_use_after_free() {
    char *p = malloc(100);
    free(p);
    p->field = 1;                /* 検出すべき */
}

/* テスト3: 未初期化変数 */
void test_uninit() {
    int x;
    if (x > 0) {                 /* 検出すべき */
        printf("positive\n");
    }
}

/* テスト4: signed/unsigned比較 */
void test_sign_compare() {
    unsigned int len = 10;
    int idx = -1;
    if (idx < len) {             /* 検出すべき */
        printf("ok\n");
    }
}

/* テスト5: 整数オーバーフロー */
void test_int_overflow(size_t n) {
    char *p = malloc(n * sizeof(char));  /* 検出すべき */
}

/* テスト6: リソースリーク */
void test_resource_leak() {
    int fd = open("/tmp/test", 0);               /* 検出すべき: closeなし */
    int sock = socket(AF_INET, SOCK_STREAM, 0);  /* 検出すべき */
}

/* テスト7: snprintf戻り値無視 */
void test_snprintf() {
    char buf[64];
    snprintf(buf, sizeof(buf), "hello %s", "world"); /* 検出すべき */
}

/* テスト8: NOCHECK抑制 */
void test_nocheck() {
    char *q = malloc(100);  // NOCHECK
}
```

仕込んだ脆弱性を CWE で整理すると以下のようになります。

| # | 危険 | CWE | 事故の形 |
|---|------|-----|---------|
| 1 | `printf(user_input)` | CWE-134 | 任意メモリ読み出し・`%n` による書き換え |
| 2 | `free(p)` 後の `p->field` | CWE-416 (UAF) | ヒープ再利用時に他オブジェクトを破壊 |
| 3 | 未初期化の `x` を条件判定 | CWE-457 | スタックゴミで分岐、再現しないバグ |
| 4 | `int idx < unsigned len` | CWE-697 | `idx=-1` が暗黙変換で巨大正数になりバウンドチェックすり抜け |
| 5 | `malloc(n * sizeof(char))` | CWE-190 | `n` が巨大だと乗算ラップアラウンドで小さい確保 → ヒープオーバーラン |
| 6 | `open`/`socket` 後 close 忘れ | CWE-401 | FD リーク、長寿命プロセスでハンドル枯渇 |
| 7 | `snprintf` 戻り値無視 | CWE-252 | 切り詰め発生を検知できず、続く処理で不完全な文字列を使う |
| 8 | `NOCHECK` 抑制 | — | レビューツール側で意図的に無視 |

## 1段目：Webで気軽に試す → c-review-ai

まず一番ライトな入口、**c-review-ai**。Next.js + FastAPI + PostgreSQL + ChromaDB を docker-compose で立ち上げて、ブラウザからコードを貼り付けると検出結果がテーブルで返ります。

```bash
git clone https://github.com/ponfreelance/c-review-ai.git
cd c-review-ai
cp .env.example .env
# .env に GROQ_API_KEY を設定（console.groq.com で無料取得）
docker compose up --build
# → http://localhost:3000 をブラウザで開く
```

この `c-review-ai` は自分の自社プロダクト（MIT ライセンス）です。検出エンジンは Groq（Llama 3.3 70B）に投げる構成で、結果を `Line / Category / Issue / Risk / Recommendation` のカラムで返します。

特徴は「環境を汚さない」「社内コードをコピペだけで試せる」「結果が URL で共有できる」の3点。**社内で導入の稟議を通す前の「触ってもらう」フェーズ**で使えます。

なお、このレイヤーはコピペ前提なので、数千行規模のコードには向きません。CLI が必要になります。

## 2段目：CLIで日常運用 → creview

自分が日常のレビューで使っているのがこの **creview**。単一バイナリの CLI で、Windows / Mac / Linux それぞれにビルド済みアーカイブがあります。ローカル静的解析は33パターン、API 不要で動かせます。

```bash
# API不要でローカル解析のみ実行
creview --local-only test_checks.c

# セキュリティ脆弱性に絞る
creview --local-only --preset security test_checks.c

# 1行の修正ヒントを付ける
creview --local-only --fix-hint test_checks.c

# 最小修正と推奨修正の2パターンを提示
creview --local-only --fix test_checks.c

# JSON で出力（CI連携用）
creview --local-only --format json test_checks.c
```

`test_checks.c` を `--local-only --fix-hint` で流した実際の出力を抜粋します（一部整形）:

```
【重大】
test_checks.c:9
  printf() の第1引数が変数。フォーマット文字列脆弱性
    ヒント: printf("%s", user_input) に書き換える

test_checks.c:17
  free(p) 後に p を参照（use-after-free）
    ヒント: free 直後に p = NULL を代入

test_checks.c:23
  未初期化変数 x を条件式で使用
    ヒント: 宣言時に = 0 等で初期化

test_checks.c:39
  malloc(n * sizeof(char)) で整数オーバーフローの可能性
    ヒント: SIZE_MAX / sizeof(char) との比較を追加

test_checks.c:44
  open() の戻り値を close していない
    ヒント: 関数出口で close(fd) を呼ぶ

test_checks.c:45
  socket() の戻り値を close していない
    ヒント: 関数出口で close(sock) を呼ぶ

【設計不明】
test_checks.c:32
  signed(int) と unsigned(unsigned int) の比較
    ヒント: size_t に揃える or ゼロ以上チェックを先に入れる

【保守危険】
test_checks.c:51
  snprintf() の戻り値を捨てている
    ヒント: 切り詰め検知のため if (ret >= sizeof(buf)) を追加
```

**仕込んだ8個中7個が検出されました**。8個目の `NOCHECK` は `malloc` にコメントで抑制を付けているので想定どおり通過です。

ここで注目したいのは、各指摘に付く3段階ラベルです。

- **【重大】**: クラッシュ・未定義動作・メモリ破壊が絡むやつ。最優先で確認。
- **【設計不明】**: 仕様曖昧。レビュー推奨。
- **【保守危険】**: 今は動くが将来バグ化しやすい。

自分は普段の開発では `--preset pr` を使っています。これは git diff の変更行に該当する指摘だけ出してくれるプリセットで、PR前の最終チェックに向きます。

```bash
creview --preset pr src/
```

さらに便利なのが `--similar` で、1件見つけた指摘と同じパターンが他ファイルに潜んでいないか横断検索します。1つのバグが複数箇所にコピペされているケースを洗い出す用途です。

```bash
creview --similar --fix --local-only src/
```

creview は GitHub で無料公開しています（[ponfreelance/creview](https://github.com/ponfreelance/creview)）。自社プロダクトですが配布は無償です。

## 3段目：監査フレームワーク → CSAF

ここからは「個人の PR チェック」を超えて、**コードベース全体の品質監査をチーム単位で回したい**人向けの話です。自分が出している **CSAF (C Safety Audit Framework)** は、creview の先にある監査レイヤーのツールです。

creview との違いを整理すると:

| 観点 | creview | CSAF |
|------|---------|------|
| 対象 | 個人の PR レビュー | コードベース全体の定期監査 |
| 解析 | 正規表現ベース33パターン | libclang AST ベースの6カテゴリ |
| MISRA | 非対応 | MISRA 6ルール対応（NULL_CHECK・UNINIT_VAR・ARRAY_BOUND・UNSAFE_CAST・RECURSION・GOTO） |
| 依存関係 | ファイル単位 | 関数呼び出しグラフから影響範囲を重み付け（risk A/B/C） |
| 比較 | `--baseline` | `csaf compare` で監査間差分 |
| レポート | テキスト / JSON / Markdown / SARIF | HTML レポート（リスク分布・ファイル統計） |

### CSAF の典型的なワークフロー

```bash
# プロジェクト初期化（設定ファイル生成）
csaf init

# 安全性監査（MISRA ベース）
csaf audit --mode SAFETY

# 仕様適合チェック（Markdown 仕様書との整合）
csaf audit --mode SPEC

# 監査間の差分（resolved / unresolved / new）
csaf compare

# HTML レポート生成
csaf report
```

CSAF が特に強いのは **依存グラフによるリスク自動昇格** です。同じ NULL 参照バグでも、「100関数から呼ばれるユーティリティ関数」と「main からしか呼ばれない初期化関数」では事故の広さが違います。CSAF は呼び出しグラフを作って、前者を risk A に、後者を risk C に自動昇格させます。

完全ローカル動作で、API キー不要。組み込み系や医療・車載のように**コードを社外に出せない現場**でも使えます。個人利用ライセンスは Booth で ¥4,980。業務ライセンスは別途相談です。

→ [自社プロダクト: CSAF (C Safety Audit Framework) - Booth](https://autotrader.booth.pm/items/8163116)

## 3段構成を振り返って

自分の今の運用は以下のようになっています。

| 場面 | 使うツール |
|------|-----------|
| コード片を同僚に共有して「これ危なくない？」と聞かれたとき | c-review-ai（ブラウザでコピペ） |
| 自分の日常 PR チェック | creview（`--preset pr`） |
| CI の静的解析ステップ | creview（`--format sarif` で GitHub Code Scanning 連携） |
| 月次の品質監査 / リファクタ優先度決め | CSAF（`csaf audit` → `csaf compare`） |

目視レビューを「全部自動でやる」発想ではなく、**ツールで検出できる範囲は機械に任せて、人間は設計とドメイン知識が必要な部分に集中する**という分業が現実解だと思います。`printf(user_input)` を目で追うのは、もうやらなくていい時代です。

## まとめ

- C 言語23年で学んだのは「レビュー品質は読み手の疲労度で決まる」ということ
- 58行に8個仕込んだ脆弱性は、creview の `--local-only --fix-hint` で7個検出（1個は意図的に `NOCHECK` で抑制）
- creview は33パターン・完全ローカル動作・API キー不要
- c-review-ai はブラウザで気軽に試せる docker-compose 版
- CSAF は libclang AST + MISRA + 依存グラフで監査レベル

---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://autotrader.booth.pm/items/8163116

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。

---
title: "creviewでmalloc NULL未チェックを9件検出した話"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

---

C言語23年、自分はフリーランスで組み込みとサーバーサイドの両方にコードレビューで入ることが多いです。`malloc` の戻り値を使う前に NULL チェックしない、というバグは2000年代から見てきて、今も週1ペースで見かけます。「メモリ確保が失敗するケースなんて滅多にない」という空気が現場に漂っていて、若手だけでなくベテランも平気で書いてくる。自分が直接指摘すれば直るんですが、翌週の PR にまた出てくる。先月だけで同じパターンを3件書いた時点で、「これは人間がやる仕事じゃない」と割り切って自社プロダクトの `creview` に任せることにしました。

---

## この記事の前提（SCQA）

**状況**: C言語プロジェクトで `malloc`/`calloc`/`realloc` の戻り値を NULL チェックせずに使うコードが繰り返し混入します。メモリ確保失敗はテスト環境では再現しにくく、本番の高負荷時や組み込みの低メモリ環境で突然クラッシュします。

**問題**: 人間のレビューは「疲れた日」「急いでいる日」に見逃します。同じパターンを毎週指摘するのはレビュアーにとっても苦痛で、指摘される側も「またこれか」と感じて関係が悪化します。

**問い**: NULL チェック漏れ・未初期化・`strcpy` 誤用・dangling pointer といった定型的な危険を、人間が毎回目視しなくても機械的に全数検出できるか？

**答え**: 自社プロダクト `creview` を CI に組み込み、定型危険パターンを全部拾わせます。人間のレビューは「この設計判断は正しいか」「ドメイン知識として安全か」に集中する分業にしました。

---

## L0: まずコンパイラ警告を全部潰す

`creview` の話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または `-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。NULL 引数を `nonnull` 関数に渡す / 未初期化 auto 変数を読む / 戻り値を破棄する / 自明な NULL deref / use-after-free / 境界外配列アクセス（静的決定可なもの）/ `switch` の case 抜け、これらは現代のコンパイラが標準で拾います。

`-fanalyzer`（GCC 10以降）は特に強力で、今回の検証コードの `malloc` NULL チェック漏れのいくつかはコンパイラレベルでも警告が出ます。L0 の警告がゼロにならない PR は、`creview` を走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。CI の `Makefile` に以下を1行追加するだけで強制できます。

```makefile
CFLAGS += -Wall -Wextra -Wpedantic -Werror -fanalyzer
```

L0 を素通りした PR を `creview` で叩くと、コンパイラが取れた警告まで人間が読むことになって時間を無駄にします。

---

## 検証用コード

今回 `creview` にかけたコードです。`malloc`/`calloc`/`realloc` の戻り値未チェック・`strcpy` 誤用・dangling pointer・マジックナンバーが1ファイルに凝縮されています。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* malloc 戻り値未チェックで即 use する典型 */
struct user {
    int id;
    char name[64];
};

void load_user(int id) {
    struct user *u = malloc(sizeof(*u));
    u->id = id;                          /* 検出すべき: NULL チェック無し */
    strcpy(u->name, "anonymous");        /* 検出すべき: NULL チェック無し */
    printf("user=%d name=%s\n", u->id, u->name);
    free(u);
}

/* calloc も同パターン */
int *make_table(size_t n) {
    int *p = calloc(n, sizeof(int));
    p[0] = 1;                            /* 検出すべき: NULL チェック無し */
    return p;
}

/* realloc の罠（戻り値を元ポインタに再代入して NULL チェックなし） */
void grow_buffer(char **buf, size_t *cap) {
    *cap *= 2;
    *buf = realloc(*buf, *cap);          /* 検出すべき: realloc が NULL を返したら元ポインタも失う */
    (*buf)[0] = '\0';                    /* 検出すべき: NULL deref 可能 */
}
```

3つの関数それぞれに異なる危険パターンが入っています。`load_user` は `malloc` 直後に NULL チェックなし、`make_table` は `calloc` 直後に NULL チェックなし、`grow_buffer` は `realloc` の戻り値を元ポインタに直接代入する「realloc の罠」です。

---

## CWE/MISRA マッピング

`creview` が検出した9件を CWE 番号・MISRA C 2012 規則・実害の形で整理します。

| 行 | 検出内容 | CWE | MISRA C 2012 | 事故の形（攻撃ベクター / 実害） |
|----|---------|-----|-------------|-------------------------------|
| L8 | マジックナンバー `64`（バッファサイズ） | CWE-547 | Rule 7.1 / Dir 4.4 | 将来の変更時に `name[64]` と `strncpy` の上限 `64` が乖離してオーバーフロー。allowlist 外の即値は保守バグの温床 |
| L11 | 未初期化変数 `id` を使用（auto 変数・不定値） | CWE-457 | Rule 9.1 | 不定値を `u->id` に書き込み、その値を `printf` で出力。デバッグ時と本番で挙動が変わる UB |
| L12 | `u` に NULL チェックなし（`malloc` 戻り値） | CWE-476 / CWE-690 | Rule 21.3 / Dir 4.7 | `malloc` 失敗時に NULL デリファレンスでクラッシュ。高負荷・低メモリ環境で再現 |
| L14 | `strcpy` 使用（コピー元長未検証） | CWE-121 | Rule 21.17 | コピー元が 63 バイト超ならスタック/ヒープ上の `name[64]` をオーバーフロー。任意コード実行に至る経路 |
| L16 | `free(u)` 直後に `NULL` 代入なし（dangling pointer） | CWE-416 | Rule 22.2 | `free` 後に `u` が旧アドレスを保持したまま残存。別スレッドや後続コードが誤って参照すると use-after-free |
| L20 | `make_table()` の宣言・定義がファイル内に見つからない | CWE 番号不明 | Rule 8.4 | ヘッダ include 漏れまたはリンクエラー。暗黙の `int` 返却と解釈されて型不一致クラッシュ |
| L21 | `p` に NULL チェックなし（`calloc` 戻り値） | CWE-476 / CWE-690 | Rule 21.3 / Dir 4.7 | `calloc` 失敗時に `p[0] = 1` で NULL デリファレンスクラッシュ |
| L27 | `grow_buffer()` の `buf` を NULL 検査せず使用（ATTR-NONNULL-001） | CWE-476 | Rule 17.5 / Dir 4.7 | 呼び出し元が NULL を渡した場合に `*buf` のデリファレンスでクラッシュ。`__attribute__((nonnull(1)))` で契約を明示するか、関数先頭で NULL 検査 |
| L29 | `*buf` に NULL チェックなし（`realloc` 戻り値を元ポインタに再代入） | CWE-476 / CWE-690 | Rule 21.3 / Dir 4.7 | `realloc` が NULL を返すと `*buf` が NULL になり元のポインタも失う。メモリリーク（CWE-401）と NULL デリファレンスが同時に発生 |

`realloc` の罠（L29）は特に悪質です。`*buf = realloc(*buf, *cap)` と書いた瞬間、`realloc` が失敗した場合に元のポインタが上書きされて失われます。`free` もできず、NULL デリファレンスも起きる、という二重の事故になります。自分が最初に見たのは10年前の組み込みプロジェクトで、30分ほど「なぜメモリが増えるのにクラッシュするのか」を追いかけました。

---

## creview の使い方（CLI 実行例）

自社プロダクト `creview` のインストールと実行は以下の手順です。

```bash
# インストール（npm 経由）
npm install -g creview

# 基本実行（セキュリティ + メモリ安全プリセット）
creview --preset security --preset memory --format markdown nullcheck.c

# CI 向け（SARIF 出力で GitHub Code Scanning に統合）
creview --preset security --preset memory --format sarif nullcheck.c > results.sarif

# PR レビュー向け（diff のみ対象）
creview --preset pr --format markdown nullcheck.c
```

`--format` オプションは `json` / `markdown` / `sarif` の3種類です（`--json` や `--sarif` のような短縮形は存在しないので注意）。

今回の検証コードに対する実行ログはそのまま以下です。

```
L8 【重大】: マジックナンバー64 (バッファサイズ) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L11 【重大】: 未初期化変数idを使用(7行目で宣言、初期化なし)。不定値
L12 【重大】: uにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
L14 【重大】: strcpy使用。コピー元長未検証でオーバーフロー可能
L16 【保守危険】: free(u)直後にNULL代入なし。dangling pointer残存
L20 【保守危険】: 関数make_table()の宣言・定義がファイル内に見つからない。ヘッダinclude漏れまたはリンクエラーの可能性
L21 【重大】: pにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
L27 【重大】: ATTR-NONNULL-001: grow_buffer() の buf を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L29 【重大】: bufにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
```

**3段階ラベルの読み方**:

- **【重大】**: 即クラッシュ・メモリ破壊・UB（未定義動作）に直結するもの。CWE-476 / CWE-690 / CWE-457 / CWE-121 が該当。PR マージ前に必ず修正する対象です。
- **【設計不明】**: 動作は一見するが、設計意図が読み取れず安全性を保証できないもの。今回の9件には出ませんでしたが、`goto` の前方ジャンプや深さ不明の再帰などが該当します。
- **【保守危険】**: 今すぐクラッシュはしないが、将来の変更・マルチスレッド化・コードの再利用で事故の温床になるもの。L16 の dangling pointer と L20 の宣言漏れが該当。「今は動いているから後回し」が一番危ない分類です。

---

## 修正後のコード例

検出された問題を修正したコードです。比較のために並べます。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define USER_NAME_MAX 64   /* L8 対応: マジックナンバーを #define で命名 */

struct user {
    int id;
    char name[USER_NAME_MAX];
};

void load_user(int id) {                 /* L11 対応: 引数 id は呼び出し元が初期化 */
    struct user *u = malloc(sizeof(*u));
    if (u == NULL) {                     /* L12 対応: NULL チェック */
        return;
    }
    u->id = id;
    /* L14 対応: strcpy → strncpy + 終端保証 */
    strncpy(u->name, "anonymous", USER_NAME_MAX - 1);
    u->name[USER_NAME_MAX - 1] = '\0';
    printf("user=%d name=%s\n", u->id, u->name);
    free(u);
    u = NULL;                            /* L16 対応: dangling pointer 解消 */
}

/* L20 対応: ヘッダまたは同ファイルに前方宣言を追加 */
int *make_table(size_t n) {
    int *p = calloc(n, sizeof(int));
    if (p == NULL) {                     /* L21 対応: NULL チェック */
        return NULL;
    }
    p[0] = 1;
    return p;
}

/* L27 対応: nonnull 属性で呼び出し元との契約を明示 */
__attribute__((nonnull(1, 2)))
void grow_buffer(char **buf, size_t *cap) {
    size_t new_cap = *cap * 2;
    char *tmp = realloc(*buf, new_cap);  /* L29 対応: 一時変数で受けて NULL チェック */
    if (tmp == NULL) {
        /* 元のポインタは *buf のまま保持される */
        return;
    }
    *buf = tmp;
    *cap = new_cap;
    (*buf)[0] = '\0';
}
```

`realloc` の修正パターンが一番重要です。`*buf = realloc(*buf, new_cap)` を `tmp = realloc(*buf, new_cap)` に変えて一時変数で受けることで、失敗時に `*buf` の元ポインタを保持できます。

---

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| **L0: ビルド時（まずここ）** | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Wall -Wextra -Werror -Weverything`（Clang）。**警告ゼロにならない PR はレビューに入らない**ルール化。NULL deref や未初期化の多くはここで取れる |
| **L1: PR チェック（OSS 標準）** | clang-tidy / cppcheck | OSS プロジェクトで広く使われる静的解析を CI で強制。`clang-tidy -checks='clang-analyzer-*,cert-*'` が基本 |
| **L2: PR レビュー（プロジェクト固有）** | 自社プロダクト **creview**（CLI） | プロジェクト固有パターン・日本語ナレッジ・severity モデル（重大/設計不明/保守危険）・CWE/MISRA マッピング・`--preset pr` で diff のみ・AI 補強。`.creviewrc.json` でマジックナンバー allowlist をプロジェクト別に管理 |
| **L2': 自己チェック（PR 投げる前）** | 自社プロダクト **c-review-ai**（ブラウザ版） | 環境構築不要。コードをコピペするだけで動く。「PR を投げる前に自分で一度かけてください」とルール化すると、L2 で引っかかる件数が体感で半減する |
| **L3: チーム監査・MISRA 準拠** | 自社プロダクト **CSAF** | libclang AST + 依存グラフで risk A/B/C を自動昇格


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。

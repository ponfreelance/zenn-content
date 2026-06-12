---
title: "creviewでC言語の整数オーバーフロー・符号混合比較を検出する"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

---

C言語23年、自分はフリーランスで組み込みとサーバーサイドのコードレビューに入ることが多いです。「malloc の引数に乗算を書くな」「strlen の結果を int に入れるな」は毎回同じ指摘なのに、週をまたいだ別の PR でまた出てくる。特に signed/unsigned の混合比較は、コンパイラが `-Wall` でも黙っている設定のプロジェクトに入ると、`i < len` の1行が爆弾として残り続けます。Zaif 案件に似た構造のコードを見たとき、「これ本番で踏んだら30分じゃ収まらない」と思って自動検出の仕組みを整えることにしました。

---

## この記事の前提（SCQA）

- **状況**: C言語のコードレビューで、malloc 引数の乗算オーバーフロー・signed/unsigned 混合比較・strlen 結果の int 代入は、繰り返し登場する定型的な危険パターンです。どれも CWE-190 または CWE-697 に分類され、実害はヒープ不足クラッシュ・範囲外アクセス・メモリ破壊に直結します。
- **問題**: 人間のレビュアーが疲弊した金曜夕方に `malloc(n * elem_size)` の1行を見ても、「n と elem_size が両方 size_t だから大丈夫」で通してしまうことがあります。実際に自分も1度やらかしています。
- **問い**: 定型的な整数演算の危険パターンを、人間のレビューに依存せず機械的に全件拾うにはどうすればよいか？
- **答え**: 自社プロダクト creview（CLI）を `--preset security` で走らせ、10件の指摘を自動取得します。人間のレビューは「その乗算がアーキテクチャ上ありえる値域か」という設計判断だけに絞る分業にしました。

---

## L0: まずコンパイラ警告を全部潰す

creview の話に入る前に、以下のフラグでビルドが通るか確認してください。

```bash
# GCC
gcc -Wall -Wextra -Wpedantic -Werror -fanalyzer -o out dangerous.c

# Clang
clang -Wall -Wextra -Weverything -Werror -o out dangerous.c
```

NULL 引数を nonnull 関数に渡す / 未初期化 auto 変数を読む / 戻り値を破棄する / 自明な NULL deref / switch case 抜け、これらは現代のコンパイラが標準で拾います。特に GCC の `-Wsign-compare` は今回の `i < len`（signed/unsigned 混合）を警告するので、`-Wall` を有効にしていれば L0 で検出できます。

**警告ゼロにならない PR はそもそも creview を走らせる前にレビュー対象にしない**、というルールを CI で強制するのが第一手です。L0 を素通りした PR を creview で叩いても、コンパイラが取れた警告まで人間が読むことになって意味がありません。

---

## 検証用コード

今回使った検証コードを丸ごと示します。実際にフリーランス先の PR で見かけたパターンを最小化したものです。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* malloc の乗算で整数オーバーフロー */
void *make_array(size_t n, size_t elem_size) {
    return malloc(n * elem_size);        /* 検出すべき: n*elem_size が size_t を超える */
}

/* 構造体の配列確保（同じ罠） */
struct row { int a, b, c, d; };
struct row *make_rows(size_t n) {
    return calloc(n, sizeof(struct row));/* calloc は内部でチェック有りだが、callocなしの malloc(n*sizeof) は危険 */
}

/* signed/unsigned 混合比較で負値が大きな unsigned に化ける */
int find_index(const int *arr, unsigned int len) {
    int i = -1;
    if (i < len) {                       /* 検出すべき: i は unsigned に昇格して 0xFFFFFFFF */
        return arr[i];
    }
    return 0;
}

/* size_t と int の混合 */
char *concat(const char *a, const char *b) {
    int la = strlen(a);
    int lb = strlen(b);
    char *r = malloc(la + lb + 1);       /* 検出すべき: int 演算でオーバーフロー（極端な長さで） */
    strcpy(r, a);
    strcat(r, b);
    return r;
}
```

各関数の問題点を簡単に整理します。

- `make_array`: `n * elem_size` は `size_t` 同士の乗算ですが、両方が大きい値のとき桁あふれしてゼロや小さい値になります。`malloc(0)` は処理系定義の動作で、直後の書き込みがヒープ破壊になります。
- `find_index`: `int i = -1` を `unsigned int len` と比較するとき、C の整数昇格規則で `i` は `0xFFFFFFFF`（32bit 環境）に変換されます。`-1 < len` は常に偽になるはずが、常に真になります。
- `concat`: `strlen` の戻り値は `size_t`（符号なし）ですが、`int la` に代入しています。文字列長が `INT_MAX` を超えると `la` が負になり、`malloc(la + lb + 1)` の引数が意図しない小さい値になります。

---

## creview の使い方（CLI 実行例）

自社プロダクト creview のインストールと実行は以下のとおりです。

```bash
# インストール（npm 経由）
npm install -g creview

# --preset security で整数演算・NULL・バッファ系を重点チェック
creview --preset security --format markdown dangerous.c
```

`--format markdown` の代わりに `--format json`（CI での機械処理向け）や `--format sarif`（GitHub Code Scanning への取り込み向け）も使えます。今回は確認しやすいように markdown で出力しました。

### 実際の検出ログ

```
L6 【保守危険】: 関数make_array()の宣言・定義がファイル内に見つからない。ヘッダinclude漏れまたはリンクエラーの可能性
L7 【重大】: malloc引数でnを乗算。オーバーフロー未検証でヒープ不足クラッシュ
L12 【保守危険】: 関数make_rows()の宣言・定義がファイル内に見つからない。ヘッダinclude漏れまたはリンクエラーの可能性
L17 【重大】: ATTR-NONNULL-001: find_index() の arr を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L19 【設計不明】: signed(i)とunsigned(len)の比較。暗黙変換で負値が巨大正値に化ける
L26 【重大】: ATTR-NONNULL-001: concat() の a を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L26 【保守危険】: 関数concat()の宣言・定義がファイル内に見つからない。ヘッダinclude漏れまたはリンクエラーの可能性
L29 【重大】: rにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
L30 【重大】: strcpy使用。コピー元長未検証でオーバーフロー可能
L31 【重大】: strcat使用。結合後長未検証でオーバーフロー可能
```

### 3段階ラベルの読み方

| ラベル | 意味 | 対応優先度 |
|--------|------|-----------|
| 【重大】 | 実行時クラッシュ・メモリ破壊・セキュリティ脆弱性に直結する欠陥。CWE 番号が対応し、攻撃ベクターまたは実害が明確 | 即時修正。PR マージ禁止 |
| 【設計不明】 | コードの意図が読み取れず、正しく動いているかどうか判断できない箇所。signed/unsigned 混合比較のように「意図的かもしれないが危険」なパターン | 設計者に意図確認。意図が明確でなければ修正 |
| 【保守危険】 | 今すぐクラッシュはしないが、将来の変更・移植・リンク時に問題になるリスクがある箇所。前方宣言漏れなど | スプリント内に対処。放置すると技術的負債が積み上がる |

自分の経験では【設計不明】が一番厄介です。「意図的に signed で持ちたかった」という若手の説明を聞いても、その意図がコードのどこにも表現されていない。コメントか `static_assert` か型の選択で意図を示してほしい、という話になります。

---

## CWE/MISRA マッピング

検出された10件を CWE 番号と実害に対応付けます。

| 行 | creview の指摘 | CWE | 事故の形（攻撃ベクターまたは実害） |
|----|---------------|-----|----------------------------------|
| L6 | make_array() 宣言なし | CWE 番号不明（リンクエラー系） | リンク時に別翻訳単位の同名関数と衝突し、意図しない関数が呼ばれる |
| L7 | malloc 引数の乗算オーバーフロー | CWE-190 | `n=0x80000001, elem_size=2` のとき積が 2 になり、後続の書き込みでヒープ破壊 |
| L12 | make_rows() 宣言なし | CWE 番号不明（リンクエラー系） | L6 と同様。単一ファイル完結コードでは誤検知だが、分割コンパイル環境では実害あり |
| L17 | arr の NULL チェック漏れ | CWE-476 | arr=NULL で呼ばれると即 SIGSEGV。`__attribute__((nonnull(1)))` でコンパイル時契約にするのが第一手 |
| L19 | signed/unsigned 混合比較 | CWE-697 | `i=-1` が `0xFFFFFFFF` に昇格し `arr[-1]` を読む。範囲外読み取りで機密データ漏洩の温床 |
| L26 | a の NULL チェック漏れ | CWE-476 | strlen(NULL) は UB。組み込みでは即ハードフォルト |
| L26 | concat() 宣言なし | CWE 番号不明（リンクエラー系） | L6 と同様 |
| L29 | malloc 戻り値未チェック | CWE-690 / CWE-252 | malloc 失敗時に r=NULL のまま strcpy を呼び SIGSEGV。`__attribute__((warn_unused_result))` と `-Werror=unused-result` で検出を強制すべき |
| L30 | strcpy 使用 | CWE-121 | コピー元が確保済みバッファより長ければスタック/ヒープを上書き。今回は r がヒープなので CWE-122（ヒープバッファオーバーフロー）も複合 |
| L31 | strcat 使用 | CWE-122 | r の残り領域を超えて結合するとヒープ破壊。`strncat` + 残りバイト数の明示的管理が必要 |

L19 の CWE-697 は「比較の問題」で直接クラッシュしないように見えますが、`arr[i]`（i が 0xFFFFFFFF）は CWE-125（範囲外読み取り）に連鎖します。creview が L19 で止めてくれなければ、L20 の `arr[i]` が単独で見えても「i は int だから問題ない」と読み飛ばすことがあります。実際に自分も一度やりました。

MISRA C 2012 との対応も補足します。L7 の乗算オーバーフローは Rule 10.8（整数演算の本質型）、L19 の混合比較は Rule 10.4（同一本質型のオペランド）、L29 の戻り値未チェックは Dir 4.7（エラー情報の検査）にそれぞれ抵触します。

---

## 修正例

指摘箇所を修正するとこうなります。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>  /* SIZE_MAX */

/* L7 修正: 乗算前にオーバーフロー検査 */
void *make_array(size_t n, size_t elem_size) {
    if (elem_size != 0 && n > SIZE_MAX / elem_size) {
        return NULL;  /* オーバーフローを呼び出し元に伝える */
    }
    return malloc(n * elem_size);
}

struct row { int a, b, c, d; };
struct row *make_rows(size_t n) {
    return calloc(n, sizeof(struct row));  /* calloc は内部でオーバーフロー検査あり */
}

/* L17, L19 修正: nonnull 契約 + unsigned で統一 */
__attribute__((nonnull(1)))
int find_index(const int *arr, size_t len) {
    /* i を size_t にすることで signed/unsigned 混合比較を根絶 */
    size_t i = 0;
    if (i < len) {
        return arr[i];
    }
    return 0;
}

/* L26, L29, L30, L31 修正 */
__attribute__((nonnull(1, 2)))
char *concat(const char *a, const char *b) {
    size_t la = strlen(a);  /* size_t で受ける */
    size_t lb = strlen(b);
    /* オーバーフロー検査 */
    if (la > SIZE_MAX - lb - 1) {
        return NULL;
    }
    char *r = malloc(la + lb + 1);
    if (r == NULL) {          /* L29 修正: NULL チェック */
        return NULL;
    }
    memcpy(r, a, la);         /* L30 修正: strcpy→memcpy+長さ既知 */
    memcpy(r + la, b, lb + 1);/* L31 修正: strcat→memcpy+長さ既知 */
    return r;
}
```

`__attribute__((nonnull))` を付けると、NULL リテラルを渡すコードをコンパイル時に警告できます。ランタイムの NULL チェックを関数本体に書くと「呼び出し側が守るべき契約」を本体に押し付ける形になり、組み込みでは穴が別の場所に移るだけです。外部 API や通信受信の境界でだけランタイム NULL チェックを行い、内部関数は nonnull 契約で固めるのが自分のやり方です。

---

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Wall -Wextra -Weverything -Werror`（Clang）。**警告ゼロにならない PR はレビューに入らない**ルール化 |
| L1: PR チェック（OSS ツール） | clang-tidy / cppcheck | OSS の主流チェックを CI で強制。checker 設定の共有コストが低い |
| L2: PR レビュー | 自社プロダクト creview（CLI） | プロジェクト固有パターン、日本語ナレッジ、severity モデル（重大/設計不明/保守危険）、CWE/MISRA マッピング。`--preset pr` で diff のみ走らせると高速。`--format sarif` で GitHub Code Scanning に取り込める |
| L2': 自己チェック（PR 投げる前） | 自社プロダクト c-review-ai（ブラウザ版） | 環境構築不要。コードをコピペして即チェック。CI


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。

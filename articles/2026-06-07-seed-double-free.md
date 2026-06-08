---
title: "creviewでdouble free／use-after-freeを14件検出した話"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

---

C言語23年、自分はフリーランスで組み込みとサーバーサイドのコードレビューに入ることが多いです。メモリ管理バグで一番「またこれか」となるのが double free と dangling pointer の2つで、どちらも「1回目の free は正しい」ので静的解析が難しく、人間のレビューでも見逃しやすい。実際、自分が入ったあるプロジェクトでは、エラーパスの分岐を追わないと気づかない double free が本番に流れ込み、再現性の低いクラッシュとして2週間放置されました。同じ失敗を繰り返させないために、自社プロダクト creview を使ってパターンを機械に拾わせる運用に切り替えた経緯を書きます。

---

## この記事の前提（SCQA）

- **状況**: C言語のメモリ管理バグ、特に double free（CWE-415）と use-after-free（CWE-416）は、エラーパスをまたぐ分岐でしか現れないケースが多く、コードを上から読むだけでは見落とします。自分のレビュー経験でも、「正常系は完璧なのにエラーパスだけ二重 free」という PR が月に1〜2本は来ます。
- **問題**: 人間が全分岐を追ってメモリの所有権を手動で検証するのはコストが高く、「疲れた金曜の夕方」に確実に見逃します。週をまたいだ別の PR でまた同じ指摘を書くことになり、レビューワも書かれる側も消耗します。
- **問い**: double free / dangling pointer のような「定型的だが見つけにくい」バグを、人間のレビューコストを上げずに検出するには何が現実的か？
- **答え**: 自社プロダクト creview の `--preset memory` で定型パターンを全部機械に拾わせ、人間のレビューは「所有権の設計判断」と「ドメイン固有のライフタイム仕様」に集中する分業にしました。今回はその検証コードと14件の検出結果を丸ごと公開します。

---

## L0: まずコンパイラ警告を全部潰す

creview の話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または `-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。NULL 引数を nonnull 関数に渡す / 未初期化 auto 変数を読む / 戻り値を破棄する / 自明な NULL deref / use-after-free / 境界外配列アクセス（静的決定可なもの）/ switch case 抜け、これらは現代のコンパイラが標準で拾います。

特に GCC の `-fanalyzer` は double free を追跡するパスがあり、今回の検証コードの一部はコンパイラレベルで警告が出ます。`-fanalyzer` なしで「コンパイルが通る＝安全」と思い込んでいるプロジェクトを自分は何度も見てきました。

**L0 の警告がゼロにならない PR はそもそも creview を走らせる前にレビュー対象にしない**ルールにするのが第一手です。L0 を素通りした PR を creview で叩くと、コンパイラが拾えた警告まで人間が読むことになり、本来 creview が担うべき「コンパイラでは取れない定型バグ」の検出結果が埋もれます。

---

## 検証用コード

今回の検証に使ったコードです。double free・dangling pointer・use-after-free の典型パターンを1ファイルに詰め込んでいます。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* 同じポインタを 2 回 free（double free） */
void release(char *p) {
    free(p);
    free(p);                             /* 検出すべき: 二重 free */
}

/* エラーパスで double free */
char *load_or_fail(int ok) {
    char *buf = malloc(128);
    if (!ok) {
        free(buf);
        return NULL;
    }
    free(buf);                           /* 検出すべき: 上の return NULL に行かなかった分岐で再 free */
    return buf;                          /* use-after-free + double-free 候補 */
}

/* free 後 NULL 代入を忘れた dangling pointer */
void cleanup(char **pp) {
    free(*pp);                           /* 検出すべき: *pp = NULL を入れていない */
}

/* 2 つの所有者が同じバッファを free */
void share_and_free(char *a, char *b) {
    if (a == b) {
        free(a);
        free(b);                         /* 検出すべき: 同じバッファに対する 2 回 free */
    }
}
```

このコードはコンパイル自体は通ります（`-Wall -Wextra` でも警告が出ないケースがある）。だからこそ「コンパイルが通った＝安全」の罠にはまりやすく、静的解析ツールが必要になります。

---

## CWE/MISRA マッピング

creview が検出した14件を CWE 番号と実害に対応付けます。

| 行 | 検出内容 | CWE | 事故の形（攻撃ベクター / 実害） |
|----|---------|-----|-------------------------------|
| L6 | NULL 検査なし（nonnull 属性未表明） | CWE-476 | NULL ポインタを `free()` に渡すと処理系依存の動作。glibc は abort、組み込み libc はサイレント破壊 |
| L7 | `free(p)` 直後に `p = NULL` なし | CWE-416 | dangling pointer が残り、後続コードが不定アドレスを参照する温床になる |
| L8 | `p` の double free | CWE-415 | ヒープメタデータ破壊。攻撃者が制御できる場合は任意コード実行（CVE 多数） |
| L8 | `free(p)` 直後に `p = NULL` なし | CWE-416 | 同上 |
| L12 | `load_or_fail()` の宣言がファイル内に見つからない | CWE-番号不明（リンクエラー） | 別 TU の定義と型が食い違った場合、引数の解釈がずれてサイレントバグ |
| L13 | `malloc` 戻り値の NULL チェックなし | CWE-690 | malloc 失敗時に NULL を `free()` または参照→即クラッシュ |
| L13 | マジックナンバー `128` | CWE-547 | 後から別の箇所でバッファサイズを変えたときに同期漏れ、バッファオーバーフローの遠因 |
| L15 | `free(buf)` 直後に `buf = NULL` なし | CWE-416 | dangling pointer 残存 |
| L18 | `buf` の double free（L15 で既に free 済み） | CWE-415 | ヒープメタデータ破壊。エラーパスを通らなかった分岐で再 free |
| L18 | `free(buf)` 直後に `buf = NULL` なし | CWE-416 | dangling pointer 残存、直後の `return buf` が use-after-free |
| L23 | NULL 検査なし（nonnull 属性未表明） | CWE-476 | `*pp` を dereference する前に `pp` が NULL の場合クラッシュ |
| L28 | NULL 検査なし（nonnull 属性未表明） | CWE-476 | `a == b` 比較前に `a` が NULL の場合、比較自体は UB ではないが `free(a)` でクラッシュ |
| L30 | `free(a)` 直後に `a = NULL` なし | CWE-416 | dangling pointer 残存 |
| L31 | `free(b)` 直後に `b = NULL` なし | CWE-416 | `a == b` が真の場合 `b` は `a` と同じアドレス。double free と dangling pointer が同時に発生 |

CWE-415（double free）は MISRA C 2012 Rule 22.2「動的に確保したメモリは1回だけ解放する」にも違反します。CWE-476（NULL 参照）は MISRA C 2012 Rule 17.5 / Dir 4.1 に対応します。

---

## creview の使い方（CLI 実行例）

### インストールと基本実行

```bash
# 検証コードを danger_memory.c として保存済みの前提
creview --preset memory --format markdown danger_memory.c
```

`--preset memory` は double free / use-after-free / dangling pointer / malloc 戻り値未検査に特化したルールセットを有効にします。`--format markdown` で CI のコメントに貼りやすい形式で出力されます。

### 検出ログ（実行結果そのまま）

```
L6 【重大】: ATTR-NONNULL-001: release() の p を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L7 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L8 【重大】: pの二重free。7行目で既にfree済み
L8 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L12 【保守危険】: 関数load_or_fail()の宣言・定義がファイル内に見つからない。ヘッダinclude漏れまたはリンクエラーの可能性
L13 【重大】: bufにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
L13 【重大】: マジックナンバー128 (バッファサイズ) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L15 【保守危険】: free(buf)直後にNULL代入なし。dangling pointer残存
L18 【重大】: bufの二重free。15行目で既にfree済み
L18 【保守危険】: free(buf)直後にNULL代入なし。dangling pointer残存
L23 【重大】: ATTR-NONNULL-001: cleanup() の pp を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L28 【重大】: ATTR-NONNULL-001: share_and_free() の a を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L30 【保守危険】: free(a)直後にNULL代入なし。dangling pointer残存
L31 【保守危険】: free(b)直後にNULL代入なし。dangling pointer残存
```

### 3段階ラベルの読み方

| ラベル | 意味 | 対応優先度 |
|--------|------|-----------|
| 【重大】 | クラッシュ・ヒープ破壊・任意コード実行に直結するバグ。CWE-415/416/476/690 など。PR マージ前に必ず修正 | 即修正・マージブロック |
| 【設計不明】 | コードの意図が静的解析だけでは判断できない箇所。関数の宣言が見当たらない、制御フローが追えない等。人間が設計意図を確認する必要がある | レビューワが確認 |
| 【保守危険】 | 今すぐクラッシュはしないが、後続コードが dangling pointer を踏んだ瞬間に CWE-416 に化ける。技術的負債として放置すると必ず事故になる | スプリント内に修正 |

今回の14件は【重大】が8件、【保守危険】が6件で、【設計不明】は L12 の「宣言が見つからない」が実質それに相当します。重大8件の内訳は double free が2件（L8・L18）、NULL 未検査が4件（L6・L13・L23・L28）、malloc 戻り値未検査が1件（L13）、マジックナンバーが1件（L13）です。

### SARIF 出力（CI 連携）

```bash
creview --preset memory --format sarif danger_memory.c > results.sarif
```

GitHub Actions の `github/codeql-action/upload-sarif` に渡すと、PR の diff 上にインラインでコメントが付きます。自分のプロジェクトでは `--preset pr` と組み合わせて diff のみを対象にすることで、既存コードのノイズを除いた新規追加分だけを CI でチェックしています。

---

## 3段階の使い分け

L0（コンパイラ警告）を最初に置くのが鉄則です。「creview を最初の砦」として紹介すると「それコンパイラで取れるよね？」と即ズレます。実際 Zenn のコメントで指摘されて痛い目を見たので明示します。

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| **L0: ビルド時（まずここ）** | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Wall -Wextra -Weverything -Werror`（Clang）。**警告ゼロにならない PR はレビューに入らない**ルール化。double free の一部はここで取れる |
| **L1: CI の共通チェック** | clang-tidy / cppcheck | OSS の主流チェックを CI で強制。misra-c-2012 プロファイルを cppcheck に渡すと MISRA 違反も拾える |
| **L2: PR 投げる前（自己チェック）** | 自社プロダクト c-review-ai（ブラウザ版） | 環境構築不要、コピペで試せる。「PR 投げる前に自分でチェック」の習慣化に使う。今回の検証コードを貼ると同様のパターンを検出する |
| **L2': PR レビュー（CI 組み込み）** | 自社プロダクト creview（CLI） | プロジェクト固有パターン・日本語ナレッジ・severity モデル（重大/設計不明/保守危険）・CWE/MISRA マッピング。`--preset pr` で diff のみ対象、`--format sarif` で GitHub Actions と連携 |
| **L3: チーム監査・MISRA 準拠証跡** | 自社プロダクト CSAF | libclang AST＋依存グラフで risk A/B/C 自動昇格。MISRA 準拠の証跡をレポートとして出力。車載・医療機器など第三者監査が必要な案件向け |

**運用ルール**: L0 の警告ゼロを CI の必須ゲートにする。それを通過した PR だけ creview（L2'）が走る。この順序を守らないと、コンパイラが取れる警告を creview のログと一緒に人間が読む羽目になり、本来の「定型バグの自動検出」という目的が崩れます。

---

## 検出結果から読み取れること

今回の検証コードは30行強ですが、14件の指摘が出ました。内訳を整理すると傾向が見えます。

**double free（CWE-415）の2パターン**:
- `release()` の L7→L8 は「同じ関数内で連続 free」という最も単純なケース。これは `-fanalyzer` でも取れます。
- `load_or_fail()` の L15→L18 は「エラーパスの分岐を通らなかった場合に再 free」というパターンで、分岐を追わないと見えません。自分が本番で踏んだのはこちらの亜種でした。

**dangling pointer（CWE-416 の温床）の6件**:
`free()` 直後に `p = NULL` を入れていないケースが全関数に散らばっています。「free した後に NULL を代入する」は C言語の定石ですが、週をまたいだ別の PR でまた出てくる。


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。

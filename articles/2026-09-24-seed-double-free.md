---
title: "「触ると壊れる」free回りのレガシーCに静的解析を差し込む手順"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語のレガシー保守では、free/malloc周りの関数がテストもコメントもないまま10年選手になっていることがよくあります。今回の検証コードも一見単純ですが、`release()`は同じポインタを2回free、`load_or_fail()`はif分岐の抜けにdouble freeとuse-after-freeが同居、`cleanup()`はfree後のNULL代入忘れ、`share_and_free()`は2つの引数が同じバッファを指すケースを想定していません。どれも「動いているから触らない」対象になりがちなコードで、まさにこのタイプの関数にこそ静的解析を当てる価値があります。

## この記事の前提（SCQA）

- **状況**: レガシーCの保守案件では、free/mallocを扱う関数が改修対象から外され続け、誰も中身を読まないまま残っていくことがよくあります。
- **問題**: 手を入れようとしても、動作しているコードの分岐を目で追ってdouble freeやuse-after-freeを見つけるのは骨が折れます。特にエラーパスの分岐（`load_or_fail()`のif文のように、通る場合と通らない場合でfree回数が変わるケース）は見落としやすい場所です。
- **問い**: 触るのが怖いコードに対して、読み込む前提の人間のレビューと、機械的に全パスを洗い出す静的解析をどう組み合わせるか。
- **答え**: まずコンパイラ警告をゼロにした上で、creviewでdouble free / dangling pointer / NULL契約未表明のパターンを機械的に洗い出し、人間は「このfreeは本当に呼ばれる分岐なのか」という設計判断だけに集中する形にしました。

## 検証用コード

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

## L0: まずコンパイラ警告を全部潰す

creviewの話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または`-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。NULL引数をnonnull関数に渡す、未初期化auto変数を読む、戻り値を破棄する、自明なNULL deref、use-after-free、境界外配列アクセス（静的決定可なもの）、switch case抜け、これらは現代のコンパイラが標準で拾います。

実際、GCCの`-fanalyzer`は`load_or_fail()`のようなdouble freeパターンの一部を検出できることがあります。これらが警告ゼロにならないPRは、creviewを走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。L0を素通りしたコードをL2のcreviewで叩いても、コンパイラで消せる警告まで人間が読むことになり、レビューの密度が薄まります。

## CWE/MISRA マッピング

| 検出パターン | 該当行 | CWE | 事故の形 |
|------------|-------|-----|---------|
| pの二重free | L7-8 | CWE-415 | free済み領域の再解放。ヒープメタデータ破壊、任意コード実行の足がかり |
| free後NULL代入なし（dangling pointer） | L7, L8, L15, L18, L30, L31 | CWE-416（UAFの温床） | 解放済みポインタが生き残り、後続コードで参照されるとuse-after-free化 |
| bufの二重free | L15, L18 | CWE-415 | if分岐を通らなかった場合にfreeが2回実行される典型パターン |
| bufにNULLチェックなし | L13 | CWE-690（戻り値一般ではCWE-252） | malloc失敗時にNULLポインタをそのままfreeし、未定義動作 |
| マジックナンバー128 | L13 | CWE-547 | バッファサイズの意図が読めず、変更時に別箇所との不整合が生まれる |
| p/pp/aのNULL契約未表明 | L6, L23, L28 | CWE-476周辺 | 呼び出し側がNULLを渡した場合の挙動が未定義。契約が型やattributeで明示されていない |
| load_or_fail()の宣言未検出 | L12 | CWE番号不明 | ヘッダinclude漏れやリンクエラーの兆候。今回は単一ファイル内の検証のため参考情報 |

## creview の使い方（CLI 実行例）

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
L31 【保守危険】: free(b)直後にNULL代入なし。danglingpointer残存
```

3段階ラベルの読み方は次の通りです。

- **【重大】**: double free、NULLチェック漏れなど、実行時にクラッシュや脆弱性へ直結する可能性が高い指摘。マージ前に必ず潰す対象です。
- **【設計不明】**（今回は該当なしですが）: ツールがコードの意図を判断できず、人間の設計判断待ちになる指摘。仕様書やコミット履歴と突き合わせる必要があります。
- **【保守危険】**: 今すぐ壊れるわけではないが、次の改修で事故る土壌になっている指摘。dangling pointer残存やマジックナンバーがここに入ります。

正直なところ、最初にこのログを見たときL12の「load_or_fail()の宣言・定義が見つからない」は戸惑いました。同一ファイル内に定義があるのに未検出扱いになっており、単一ファイル検証時の解析範囲の癖だと理解するまで少し読み直しています。複数ファイルにまたがるプロジェクトでは、includeパスの設定を先に確認してから結果を読む必要があります。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clangの警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Weverything -Werror`（Clang）。警告ゼロにならないPRはそもそもレビューに入らないルール化 |
| L1: PRチェック | clang-tidy / cppcheck | OSSの主流チェックをCIで強制 |
| L2: PRレビュー | creview（CLI、自社プロダクト） | プロジェクト固有パターン、日本語ナレッジ、severityモデル（重大/設計不明/保守危険）、CWE/MISRAマッピング、`--preset pr`でdiffのみ検証、AI補強 |
| L2': 自己チェック（PR投げる前） | c-review-ai（ブラウザ版、自社プロダクト） | 環境構築不要、コピペで試せる軽量版。load_or_fail()のようなfree分岐を書いた直後に自分で確認できる |
| L3: チーム監査・MISRA準拠 | CSAF（自社プロダクト） | libclang AST + 依存グラフでrisk A/B/C自動昇格。レガシー全体の棚卸しに向く |

実行コマンドの例としては以下のような形になります。

```
creview --preset memory --format markdown src/legacy_mem.c
```

`--preset memory`はfree/malloc周りのパターンに寄せた検出セットで、今回のようなdouble free中心のファイルに向いています。CI組み込みなら`--format sarif`でGitHubのCode Scanning連携に流すか、レビューコメント用に`--format markdown`をPRに貼る運用が扱いやすいです。

## まとめ

- レガシーコードのfree回りは、エラーパスの分岐（`load_or_fail()`のif文）にdouble freeが隠れやすく、目視だけでは分岐の網羅が漏れます。
- free直後のNULL代入忘れ（dangling pointer）は即座にクラッシュしないため後回しにされがちですが、【保守危険】ラベルとして機械的に洗い出せます。
- NULLチェック漏れは関数本体でifを足すより、`__attribute__((nonnull(N)))`で契約をコンパイル時に明示する方が、呼び出し側の責任を後から見失いません。
- マジックナンバー128のような検出は、severityが低く見えても次の改修者が意図を読み違える火種になるため、allowlist設定で運用ルールに落とし込む価値があります。
- L0（コンパイラ警告ゼロ）を通過したコードだけをL2のcreviewに回すことで、人間のレビューは「このfreeは本当にこの分岐で呼ばれるべきか」という設計判断に絞り込めます。


---

### 試すリンク（無料で試す → 業務で監査する）

C言語の静的解析を、いきなり有料ではなく「無料で手を動かす → 日常運用 → 監査レベル」の順で用意しています。

- **まずブラウザで試す → c-review-ai**（自社プロダクト / MIT / 無料）
  貼り付けた1ファイルをその場でレビュー。導入ゼロで挙動を確かめる用。
  → https://cutt.ly/1yuTys1c
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  手元のコードをコミット前に単発チェック。CI にも組み込める。
  → https://cutt.ly/syuTpB7z
- **リポジトリ全体を監査 → CSAF**（自社プロダクト / ¥4,980 買い切り）
  MISRA 準拠チェックと CWE データフロー解析をリポジトリ横断で走らせ、監査レポート（HTML）を出力。
  レビューの属人化を止めたいチーム向けで、商用の静的解析ツールなら数十万円かかる基本機能を、個人でも入れられる価格にしています。
  → https://cutt.ly/ItKo4MPY

無料の2つで「使える」と感じたら、リポジトリ全体に監査を回すのが CSAF です。バージョンは随時更新しています。

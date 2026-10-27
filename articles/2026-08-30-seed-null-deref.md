---
title: "malloc戻り値未チェックの9件、レビューで何行見落とすか"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みのコードレビューに入ることが多いです。malloc/calloc/reallocの戻り値未チェックは「知らない人はいない」パターンです。それでも今回検証した29行のコードから、creviewは9件の指摘を出しました。件数が多いこと自体より、9件のうち何件が「NULL チェック漏れ」という同一ラベルの繰り返しで、何件がそれとは別の見落としだったかに意味があります。手動レビューだと「mallocの戻り値チェックがない」の一発を見つけた時点でレビューが終わった気になりやすく、同じ関数内の未初期化変数やマジックナンバーまで手が回らないことが実際の検出結果から見えてきます。

## この記事の前提（SCQA）

- **状況**: malloc/calloc/reallocの戻り値未チェックはC言語では基礎知識で、レビュー観点としても真っ先に挙がります。
- **問題**: 「NULLチェックがない」という指摘を1箇所見つけた時点で、レビューする側の意識がそこに集中し、同じ関数内の別の問題（未初期化変数、マジックナンバー、dangling pointer）が素通りしやすくなります。
- **問い**: 29行3関数のコードに対して、人間が「NULLチェック漏れ」以外に何件見落とす構造になっているかを、どう可視化するか。
- **答え**: creviewで全行をパターンマッチさせた結果、NULLチェック漏れ4件に加えて、未初期化変数・マジックナンバー・strcpy・dangling pointer・関数未定義参照の5件が別カテゴリで出てきました。合計9件のうち5件は「NULLチェック探し」の意識では拾えない種類の指摘です。

## L0: まずコンパイラ警告を全部潰す

creviewの話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または `-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。NULL引数をnonnull関数に渡す、未初期化auto変数を読む、戻り値を破棄する、自明なNULL deref、use-after-free、境界外配列アクセス（静的決定可なもの）、switch case抜け、これらは現代のコンパイラが標準で拾います。

今回の検証コードで言えば、`load_user`の `int id` 未初期化使用（L11）は `-Wuninitialized` / `-Wmaybe-uninitialized` の対象になり得ますし、`-fanalyzer` を付けたGCCならmallocの戻り値未チェックの一部も検出候補に入ります。これらが警告ゼロにならないPRは、creviewを走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。

## 検証用コード

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

## CWE/MISRA マッピング

| 検出パターン | 検出行 | CWE | 事故の形 |
|------------|-------|-----|---------|
| マジックナンバー64（バッファサイズ） | L8 | CWE-547 | サイズ変更時に名前で追跡できず、charバッファとの整合崩壊 |
| 未初期化変数idの使用 | L11 | CWE-457 | auto変数を初期化前に読む未定義動作。値が不定のままstruct書き込みに使われる |
| malloc戻り値未チェック | L12 | CWE-476（CWE-690系） | NULL deref。`u->id = id` の時点でクラッシュ |
| strcpy使用 | L14 | CWE-121 | コピー元長未検証によるスタック/ヒープバッファオーバーフロー |
| free後のdangling pointer残存 | L16 | CWE-416 | 解放後もポインタが有効値を保持し、以降の参照でuse-after-freeの温床になる |
| calloc戻り値未チェック | L21 | CWE-476（CWE-690系） | `p[0] = 1` がNULL deref |
| grow_bufferでbufをNULL検査せず使用 | L27 | CWE-476周辺 | 契約未表明。nonnull属性なしで呼び出し側の保証に依存 |
| realloc戻り値未チェック | L29 | CWE-476（CWE-690系） | reallocがNULLを返すと元ポインタを失い、かつNULL derefも発生 |

L20の「make_table()の宣言・定義が見つからない」はビルド構成の問題で、CWE番号不明として扱います。

## creview の使い方（CLI 実行例）

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

【重大】はCWEに直結しクラッシュ・メモリ破壊に発展する箇所です。今回は9件中7件がこのラベルで、malloc系NULLチェック漏れ4件・未初期化変数・マジックナンバー・strcpyが含まれます。【保守危険】は今すぐ落ちるわけではないが将来のバグ温床になる箇所で、danglingポインタ残存と関数定義未検出の2件です。【設計不明】は今回のログには出ていませんが、ツールが仕様意図を判断できず人間確認が必要な指摘に使われるラベルです。

実行コマンドは以下のイメージです（`--preset security` で今回のようなメモリ安全系パターンに絞れます）。

```
creview --file sample.c --preset security --format markdown
```

自分がこの結果で引っかかったのは、L20の「make_table()が見つからない」という指摘です。最初は誤検知かと思いましたが、実際にはヘッダ分割や別ファイル定義を想定した警告で、単一ファイル検証だとこの手の「呼び出しはあるが定義が見えない」系が普通に出ます。無視して読み飛ばすと、本当にリンクエラーになる構成のときに気づけません。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Weverything -Werror`（Clang）。警告ゼロにならないPRはそもそもレビューに入らないルール化 |
| L1: PRチェック | clang-tidy / cppcheck | OSSの主流チェックをCIで強制 |
| L2: PRレビュー | 自社プロダクトcreview（CLI） | プロジェクト固有パターン、severityモデル（重大/設計不明/保守危険）、CWE/MISRAマッピング、`--preset pr` でdiffのみ |
| L2': 自己チェック（PR投げる前） | 自社プロダクトc-review-ai（ブラウザ版） | 環境構築不要、コピペで試せる軽量版 |
| L3: チーム監査・MISRA準拠 | 自社プロダクトCSAF | libclang ASTと依存グラフでrisk A/B/C自動昇格 |

**運用ルール**: L0の警告がゼロにならないPRはレビューに入らない、をCIで強制します。L0を素通りしたPRをcreviewで叩いても、消せたはずの警告まで人間が読むことになって意味がありません。

## まとめ

- 9件の検出のうち4件はmalloc/calloc/reallocのNULLチェック漏れ（L12, L21, L27, L29）で、残り5件は未初期化変数・マジックナンバー・strcpy・danglingポインタ・関数未定義参照という別カテゴリでした。
- 手動レビューは「NULLチェックがあるか」で意識が止まりやすく、同じ関数内のL8（マジックナンバー）やL11（未初期化変数）は見落とし対象になりやすい行として検出結果に現れています。
- L16のdanglingポインタ残存とL20の関数未定義参照は【保守危険】ラベルで、即クラッシュではないため手動レビューの優先順位から外れがちな箇所です。
- L27のnonnull契約未表明は、関数内でのNULLチェック追加ではなく`__attribute__((nonnull(N)))`での契約明示が第一選択で、これはレビューコメントで指摘されないと気づきにくい観点です。
- L0のコンパイラ警告設定を先に固めておかないと、creviewが拾う9件のうち一部はそもそもビルド時点で拾えたはずの重複作業になります。


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

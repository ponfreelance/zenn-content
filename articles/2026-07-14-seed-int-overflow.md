---
title: "mallocの乗算オーバーフローとsigned/unsigned比較をcreviewで検出する"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みと汎用アプリの両方のコードレビューに入ることが多いです。今回はmallocの乗算オーバーフローとsigned/unsigned混合比較を静的解析ツールcreviewで検出した記録を書きます。この2つは新人が書いたコードでも、10年書いているベテランのコードでも普通に出てきます。自分も過去に`malloc(n * sizeof(struct foo))`をレビューで素通りさせて、後日別のPRで同じ関数を呼んだ側がnに巨大値を渡してヒープ破壊した経験があります。thumbnailの検証は「大丈夫だろう」で済ませてはいけない、という教訓になりました。

## SCQA

- **状況**: malloc/callocへの引数計算で乗算を使うコードは、C言語のプロジェクトでは日常的に書かれています。多くは`n`が信頼できる小さい値である前提で動いていて、そのまま本番まで行きます。
- **問題**: `n * elem_size`がsize_tの範囲を超えると、mallocは小さいサイズを確保してしまい、その後の書き込みでヒープバッファオーバーフローが起きます。さらにsigned/unsigned混合比較では、負の値がゼロ拡張／符号拡張の規則で巨大なunsignedに化けて、境界チェックが完全に無意味になります。
- **問い**: この手のオーバーフロー系の罠は、レビューアが目視で毎回見つけられるものなのか？
- **答え**: 自分の経験では無理です。目視で気づけるのは「たまたま気分が良い日」だけで、疲れている日や差分が大きいPRでは確実に見逃します。creviewに機械的なパターン検出を任せて、人間は「このnはどこから来るのか」というデータフロー設計に集中する分業にしました。

## 検証用コード

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

## L0: まずコンパイラ警告を全部潰す

creviewの話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`(GCC)または`-Wall -Wextra -Werror -Weverything`(Clang)を有効にしてビルドが通るか確認してください。今回のコードでも`-Wsign-compare`を有効にすれば`if (i < len)`のsigned/unsigned比較警告はGCC/Clangが標準で拾います。strlenの戻り値（size_t）をintに代入する箇所も`-Wconversion`で警告が出ます。

これらが警告ゼロにならないPRは、creviewを走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。実際、自分が入ったプロジェクトでは`-Wsign-compare`を後から有効化しただけで、既存コードから20件以上のsigned/unsigned比較が出てきて、そのうち3件は本当に本番影響のあるバグでした。コンパイラで取れるものをツールに二重にやらせても意味がないので、L0を素通りさせない運用が前提になります。

## CWE/MISRA マッピング

| 検出パターン | CWE | 事故の形 |
|------------|-----|---------|
| `malloc(n * elem_size)` の乗算オーバーフロー | CWE-190 | nとelem_sizeの積がsize_tを超えると、意図したサイズより小さいバッファが確保され、後続の書き込みでヒープバッファオーバーフロー（CWE-122）に繋がる |
| `if (i < len)` のsigned/unsigned混合比較 | CWE-697 | `i = -1`が`unsigned int`に暗黙変換されて`0xFFFFFFFF`になり、境界チェックが常にtrue、`arr[i]`で範囲外アクセス |
| `int la = strlen(a)` のsize_t→int縮小変換 | CWE-190 | 極端に長い文字列（2GB超）でintがオーバーフローし、`malloc(la + lb + 1)`が負値やラップアラウンドしたサイズで確保される |
| `r`のNULLチェック漏れ | CWE-690 | malloc失敗時に`strcpy(r, a)`がNULL deref、クラッシュ |
| `strcpy`/`strcat`使用 | CWE-121 | コピー元長・結合後長が未検証で、呼び出し元次第でスタック/ヒープバッファオーバーフロー |
| `arr`/`a`のNULL契約未表明 | CWE-476周辺 | 呼び出し側がNULLを渡した場合の挙動が未定義。`nonnull`属性がないと契約が暗黙のまま伝播する |

## creview の使い方（CLI 実行例）

```
$ creview review make_array.c --preset security --format markdown
```

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

3段階ラベルの読み方は以下の通りです。

- **【重大】**: 実行時にクラッシュ・メモリ破壊・情報漏洩に直結する可能性が高いもの。マージ前に必ず潰す。今回のmalloc乗算オーバーフロー、NULLチェック漏れ、strcpy/strcatはここに入ります。
- **【設計不明】**: バグかどうかは呼び出し側の契約次第で、レビューアが「これは意図的か」を確認する必要があるもの。`i < len`の比較は、lenが常に小さい値なら問題にならないケースもあるため、設計判断が必要というラベルです。
- **【保守危険】**: 即座の脆弱性ではないが、将来のメンテナンスコストやビルド構成の問題を示すもの。今回の「関数の宣言・定義が見つからない」は単一ファイル解析特有のノイズで、ヘッダを分割した実プロジェクトでは無視して良いケースが多いです。ただし本当にリンクエラーの前兆であることもあるので、CIログと合わせて確認します。

自分がこの実行結果で最初に「あれ」と思ったのは`L26`の`concat()`が重大と保守危険の両方で出てきたところです。同じ行に複数カテゴリの指摘が乗るのは仕様なのですが、初めて見たときはツールの重複バグかと10分ほど調査して時間を溶かしました。ログのL番号だけでなく、指摘IDやカテゴリタグまで見る癖をつけてからは迷わなくなりました。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer` (GCC) / `-Weverything -Werror` (Clang)。**警告ゼロにならないPRはそもそもレビューに入らない**ルール化 |
| L1: PRチェック | clang-tidy / cppcheck | OSSの主流チェックをCIで強制 |
| L2: PRレビュー | creview（CLI） | プロジェクト固有パターン、日本語ナレッジ、severityモデル（重大/設計不明/保守危険）、CWE/MISRAマッピング、`--preset pr`でdiffのみ、AI補強 |
| L2': 自己チェック（PR投げる前） | c-review-ai（ブラウザ版） | 環境構築不要、コピペで試せる軽量版 |
| L3: チーム監査・MISRA準拠 | CSAF | libclang AST + 依存グラフでrisk A/B/C自動昇格 |

**運用ルール**: 「L0の警告がゼロにならないPRはレビューに入らない」をCIで強制する。今回のsigned/unsigned比較は`-Wsign-compare`で拾えるので、これがL0を素通りしているプロジェクトはまずコンパイラフラグを見直すのが先です。L0を通過した後にcreviewの`--preset security`を通し、mallocの乗算オーバーフローのようなコンパイラが拾いにくいパターンを追加で潰す、という順序を守っています。

## まとめ

- `malloc(n * elem_size)`の乗算はCWE-190の典型で、`calloc`が使えるなら`calloc`に置き換えるのが最も安全な対策。乗算前にオーバーフロー検査を入れる場合は`n > SIZE_MAX / elem_size`のチェックを忘れない。
- signed/unsigned混合比較（CWE-697）は`-Wsign-compare`で機械的に拾えるが、既存コードで有効化するとまとまった件数が出るので、段階的に潰す計画が必要。
- strlenの戻り値（size_t）をintで受けるコード（`int la = strlen(a)`）は縮小変換の温床で、`size_t`のまま扱うか`-Wconversion`で検出する。
- NULL契約は関数本体でのチェックより先に`__attribute__((nonnull))`で明示するほうが、呼び出し側の契約違反を早期に発見できる。
- creviewの3段階ラベルは「今すぐ直すもの」「設計判断が必要なもの」「将来のリスク」を分けるためのもので、【保守危険】を全部無視するとリンクエラーの前兆を見逃すことがあるので、CIログと突き合わせる運用が安全。


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

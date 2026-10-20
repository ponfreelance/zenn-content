---
title: "double free をCIで止める：creviewをパイプラインに組む設計"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みのコードレビューに入ることが多いです。

先月入った案件で、CIにはclang-tidyが回っていたのに、double freeを含んだPRが3つマージされた後の状態を見せられました。理由は単純で、clang-tidyの結果がWarningのまま放置されて誰も見ていなかったこと。ツールを入れることと、パイプラインで機能させることは別問題だと痛感しました。この記事は「creviewというツールの紹介」ではなく、CIのどの段階に何を置けばdouble freeやdangling pointerを本番前に止められるか、という運用設計の話です。タイトルの通り、double freeを実際に検出したログを使って、CI上の配置まで踏み込みます。

## SCQA

- **状況**: static解析ツールを導入しているプロジェクトでも、double freeやdangling pointerがレビューをすり抜けて本番に流れることがあります。
- **問題**: ツールが検出していても、CIのどの段階で・誰が・どう対応するかが曖昧だと、警告は「見なかったこと」になります。実際、自分が見た案件ではclang-tidyの指摘がSlack通知止まりで、PRのマージ条件に入っていませんでした。
- **問い**: 静的解析の検出結果を「マージブロック」まで機能させるには、CIパイプラインのどこに、どの粒度のチェックを置けばいいのか。
- **答え**: L0（コンパイラ警告)をマージ必須条件にした上で、L2（creview）をPRの`--preset security`で自動実行し、重大判定を出力したらCIをfailさせる。この2段構えで、double free系のバグは自分の手元検証では全部PRの時点で止まりました。

## L0: まずコンパイラ警告を全部潰す

creviewの話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または`-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。NULL引数をnonnull関数に渡す／未初期化auto変数を読む／戻り値を破棄する／自明なNULL deref／use-after-free／境界外配列アクセス（静的決定可なもの）／switch case抜け、これらは現代のコンパイラが標準で拾います。

これらが警告ゼロにならないPRは、creviewを走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。実際、今回の検証コードを`gcc -fanalyzer -Wall -Wextra`にかけると、`load_or_fail`のdouble free候補は`-fanalyzer`が一部反応します（ただし全パス網羅ではない）。ここで止まらなかった分を、次のL2でcreviewに拾わせる構成にします。

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

## CWE/MISRA マッピング

| 検出パターン | CWE | 事故の形 |
|------------|-----|---------|
| release()内でのpの二重free | CWE-415 | 2回目のfreeでヒープメタデータ破壊、glibcなら`malloc(): double free detected`でクラッシュ、悪用されればヒープ操作の足がかりに |
| load_or_fail()の分岐によるbufの再free | CWE-415 | エラーパスと正常パスの両方でfreeが走る設計ミス。呼び出し側にfree済みポインタを返す危険もあり |
| load_or_fail()のreturn buf | CWE-416 | free済み領域を指すポインタをそのまま返却。呼び出し側が使うとuse-after-free |
| cleanup()でのfree後NULL未代入 | CWE-416 | dangling pointerが残り、後続コードで誤参照されるとUAFの温床になる |
| share_and_free()の同一バッファ二重free | CWE-415 | ポインタが違う変数名でも中身が同じアドレスなら二重free。所有権設計の欠陥 |
| release/cleanup/share_and_freeの引数NULL未検査 | CWE-476 | NULLを渡された場合の契約が本体になく、呼び出し側の責務が曖昧。`__attribute__((nonnull))`で契約を型レベルに上げるべき |
| load_or_fail()のmalloc戻り値未チェック | CWE-690 | メモリ枯渇時にNULLを素通りしてfree(NULL)自体は安全だが、後続のreturn bufでNULLを返し呼び出し側がクラッシュ |
| malloc(128)のマジックナンバー | CWE-547 | サイズ変更時の目視漏れ、他箇所の128と意味が違うのに揃えて直され破綻するリスク |

## creview の使い方（CLI 実行例）

```
$ creview scan release.c --preset security --format markdown
```

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

L12の「宣言・定義が見つからない」は今回単一ファイル切り出しで検証したための誤検知で、実プロジェクトではヘッダに関数プロトタイプがあれば消えます。これに気づかず「なぜload_or_failだけ拾われるんだ」と原因探しをしたのが今回のハマり所です。単一ファイルでの動作確認と実プロジェクトでの結果は差が出るので、CI設定時は実際のビルド単位でスキャン範囲を決めてください。

3段階ラベルの読み方は次の通りです。

- **【重大】**: double free・NULL未検査・マジックナンバーなど、実害に直結するか設計契約が欠落しているもの。CIをfailさせる対象。
- **【設計不明】**: 今回のログには出ていませんが、責務や意図が読み取れず人間判断が必要なもの。creviewが自動判定を避けて人に投げてくるカテゴリ。
- **【保守危険】**: dangling pointer残存など、今すぐ落ちるわけではないが将来のバグの温床になるもの。マージブロックまではしないがレビューコメントで指摘する運用が現実的です。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）／`-Weverything -Werror`（Clang）。警告ゼロにならないPRはそもそもレビューに入らないルール化 |
| L1: PRチェック | clang-tidy / cppcheck | OSSの主流チェックをCIで強制 |
| L2: PRレビュー | creview（CLI、自社プロダクト） | プロジェクト固有パターン、日本語ナレッジ、severityモデル（重大/設計不明/保守危険）、CWE/MISRAマッピング、`--preset pr`でdiffのみ、AI補強 |
| L2': 自己チェック（PR投げる前） | c-review-ai（ブラウザ版、自社プロダクト） | 環境構築不要、コピペで試せる軽量版 |
| L3: チーム監査・MISRA準拠 | CSAF（自社プロダクト） | libclang ASTと依存グラフでrisk A/B/C自動昇格 |

**運用ルール**: 「L0の警告がゼロにならないPRはレビューに入らない」をCIで強制する。今回の案件でL0が形骸化していたのは、`-Werror`をつけていなかったことに尽きます。警告が出てもビルドは通るので、誰も直さないまま積もっていました。L0をブロッキングにした上でL2の`--preset security`をCIジョブに追加し、【重大】判定が1件でもあればexit code非ゼロで返す設定にしてから、double free系はPRの段階で止まるようになりました。

## まとめ

- double free（CWE-415）とdangling pointer（CWE-416の温床）は、freeの直後にNULL代入する運用ルールだけで大半が防げる
- clang-tidyやcppcheckを入れているだけでは不十分で、CIのどこでfailさせるかまで設計しないと警告は形骸化する
- creviewの`--preset security`は【重大】判定をexit codeに反映できるので、CIのマージブロック条件に直結させられる
- 単一ファイル切り出しでの検証結果と実プロジェクトでの検出結果は差が出るため、スキャン範囲はビルド単位に合わせて確認する
- L0（コンパイラ警告ゼロ）を前提条件にしないと、L2以降のツールが拾う対象がノイズだらけになる


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
